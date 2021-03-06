---
title: "NET Dynamically Generating Classes in Runtime"
date: 2017-03-23
type: post
tags: [.net,dlr,dynamic,assembly,programming]
image: "/images/dynamic-assembly.png"
---

Reflection API in .NET is one of the most powerful and incredible features of it. And along with [DLR](https://en.wikipedia.org/wiki/Dynamic_Language_Runtime) it enables a whole new look at what can be done in .NET languages.  
One of problems i had to solve recently was how to create a unified [DSL](https://en.wikipedia.org/wiki/Domain-specific_language) for queries that could be used with most C# DB drivers(they had to have [IQueryable](https://msdn.microsoft.com/en-us/library/system.linq.iqueryable(v=vs.110).aspx) support).  
It turned out to be simpler job than i thought. This [github project](https://github.com/kahanu/System.Linq.Dynamic) helped me a lot to figure out how to solve it. One of the problems was how to execute *projections* from dbs and be able to get the information in runtime.

## Overview

Solution consists mainly from 4 parts:

  * An abstract class that serves as a parent for further generation
  * A class that serves as an information about properties that will represent data fields
  * A class that's used as a *signature* for specified assemblies, since we're caching them(it will be explained why later)
  * A singleton factory class that generates, caches and returns dynamic assemblies

### Abstract class

It's just a regular empty abstract class that serves as a base for further assemblies.

```csharp
public abstract class AnonymousClass
{

}
```

### Class for information about data fields

```csharp
public class AnonymousProperty
{
    private string name;
    private Type type;

    public AnonymousProperty(string name, Type type)
    {
        if (String.IsNullOrWhiteSpace(name)) throw new ArgumentNullException(nameof(name));
        if (type == default(Type)) throw new ArgumentNullException(nameof(type));

        this.name = name;
        this.type = type;
    }

    public string Name => this.name;

    public Type Type => this.type;
}
```

### Signature class

A signature class is basically used as a *unique* hash of dynamically generated assemblies. It takes property names and property types from a generated assembly and [XORs](https://en.wikipedia.org/wiki/Exclusive_or) their hash codes with it's own hash code.

```csharp
public class AnonymousClassSignature : IEquatable<AnonymousClassSignature>
{
    private AnonymousProperty[] properties;
    private int hashCode = 0;

    public AnonymousClassSignature(IEnumerable<AnonymousProperty> properties)
    {
        if (properties == null) throw new ArgumentNullException(nameof(properties));

        this.properties = properties.ToArray();
        foreach (var property in this.properties)
        {
            this.hashCode = this.hashCode ^ property.Name.GetHashCode() ^ property.Type.GetHashCode();
        }
    }

    public override int GetHashCode() => this.hashCode;

    public override bool Equals(object obj) => obj is AnonymousClassSignature ? this.Equals(obj as AnonymousClassSignature) : false;

    public bool Equals(AnonymousClassSignature other)
    {
        if (other == null) throw new ArgumentNullException(nameof(other));
        if (this.properties.Length != other.properties.Length) return false;

        for (var i = 0; i < this.properties.Length; i++)
        {
            if (this.properties[i].Name != other.properties[i].Name || this.properties[i].Type != other.properties[i].Type)
                return false;
        }

        return true;
    }
}
```

### Singleton factory class

This is a long one. It's a lazy initialized singleton class that serves as a factory generator of dynamicly created classes and it also holds a cache of already created ones. Caching is needed because dynamically generating a class has a huge performance hit. It uses both a [ConcurrentDictionary](https://msdn.microsoft.com/en-us/library/dd287191(v=vs.110).aspx) and [ReaderWriterLockSlim](https://msdn.microsoft.com/en-us/library/system.threading.readerwriterlockslim(v=vs.110).aspx) to synchronize access to generation and retrieval of dynamic classes.  
Steps to get a dynamic class basically goes like this:

* Create a list of properties you want a dynamic class to have, with a name and type
* Give it to the factory
* Factory creates a unique signature out of those properties and checks if it already exists in cache
* If it exists retrieve it from cache
* If not, use [TypeBuilder](https://msdn.microsoft.com/en-us/library/system.reflection.emit.typebuilder(v=vs.110).aspx) to create those properties, with backing fields and get_ , set_ methods for each one, cache it and return to the caller

[ReflectionPermission](https://msdn.microsoft.com/en-us/library/system.security.permissions.reflectionpermission(v=vs.110).aspx) is because we need to generate private fields and that is not allowed by default, and basically any action that uses [Reflection.Emit](https://msdn.microsoft.com/en-us/library/system.reflection.emit(v=vs.110).aspx) requires it.

[TypeBuilder](https://msdn.microsoft.com/en-us/library/system.reflection.emit.typebuilder(v=vs.110)) is basically just a wrapper around [CIL](https://en.wikipedia.org/wiki/Common_Intermediate_Language) and currently it's the only way to have dynamically generated classes.

```csharp
public class AnonymousAssemblyFactory
{
    private static readonly Lazy<AnonymousAssemblyFactory> Lazy = new Lazy<AnonymousAssemblyFactory>(() => new AnonymousAssemblyFactory(), LazyThreadSafetyMode.ExecutionAndPublication);

    private ModuleBuilder moduleBinder;
    private ConcurrentDictionary<AnonymousClassSignature, Type> classes;
    private long classCount;
    private ReaderWriterLockSlim readerWriterLockSlim;

    private AnonymousAssemblyFactory()
    {
        var assemblyName = new AssemblyName("SomeDll.AnonymousClasses");
        var assemblyBuilder = AppDomain.CurrentDomain.DefineDynamicAssembly(assemblyName, AssemblyBuilderAccess.RunAndCollect);

#if ENABLE_LINQ_PARTIAL_TRUST
        new ReflectionPermission(PermissionState.Unrestricted).Assert();
#endif
        try
        {
            this.moduleBinder = assemblyBuilder.DefineDynamicModule("SomeDll.DynamicModule");
        }
        finally
        {
#if ENABLE_LINQ_PARTIAL_TRUST
            PermissionSet.RevertAssert();
#endif
        }

        this.classes = new ConcurrentDictionary<AnonymousClassSignature, Type>();
        this.classCount = 1;
        this.readerWriterLockSlim = new ReaderWriterLockSlim();
    }

    public static AnonymousAssemblyFactory Instance => Lazy.Value;

    public Type GetAnonymousClass(IEnumerable<AnonymousProperty> properties)
    {
        if (properties == null) throw new ArgumentNullException(nameof(properties));

        this.readerWriterLockSlim.EnterUpgradeableReadLock();

        try
        {
            var signature = new AnonymousClassSignature(properties);
            Type outType;

            if (!this.classes.TryGetValue(signature, out outType))
            {
#if ENABLE_LINQ_PARTIAL_TRUST
        new ReflectionPermission(PermissionState.Unrestricted).Assert();
#endif
                this.readerWriterLockSlim.EnterWriteLock();
                if (!this.classes.TryGetValue(signature, out outType))
                {
                    var className = $"SomeDll_AnonymousClass_{this.classCount++}";
                    var typeBuilder = this.moduleBinder.DefineType(className, TypeAttributes.Class | TypeAttributes.Public, typeof(AnonymousClass));
                    List<FieldInfo> fields;
                    this.CreateAnonymousProperties(typeBuilder, properties.ToList(), out fields);
                    this.CreateEqualsMethod(typeBuilder, fields);
                    this.CreateGetHashCodeMethod(typeBuilder, fields);
                    var result = typeBuilder.CreateType();
                    this.classes.TryAdd(signature, result);
                    Interlocked.CompareExchange(ref this.classCount, this.classes.Count, this.classes.Count);
                    return result;
                }

                return outType;
            }
            else
                return outType;
        }
        finally
        {
#if ENABLE_LINQ_PARTIAL_TRUST
            PermissionSet.RevertAssert();
#endif

            if (this.readerWriterLockSlim.IsWriteLockHeld)
            {
                this.readerWriterLockSlim.ExitWriteLock();
            }

            this.readerWriterLockSlim.ExitUpgradeableReadLock();
        }
    }

    private void CreateAnonymousProperties(TypeBuilder typeBuilder, List<AnonymousProperty> properties, out List<FieldInfo> fields)
    {
        if (typeBuilder == null) throw new ArgumentNullException(nameof(typeBuilder));
        if (properties == null) throw new ArgumentNullException(nameof(properties));

        fields = new List<FieldInfo>(properties.Count);

        foreach (var anonProperty in properties)
        {
            var field = typeBuilder.DefineField($"_{anonProperty.Name}", anonProperty.Type, FieldAttributes.Private);
            var property = typeBuilder.DefineProperty(anonProperty.Name, PropertyAttributes.HasDefault, anonProperty.Type, null);
            var getter = typeBuilder.DefineMethod($"get_{anonProperty.Name}", MethodAttributes.Public | MethodAttributes.SpecialName | MethodAttributes.HideBySig, null, new[] { anonProperty.Type });
            var setter = typeBuilder.DefineMethod($"set_{anonProperty.Name}", MethodAttributes.Public | MethodAttributes.SpecialName | MethodAttributes.HideBySig, null, new[] { anonProperty.Type });

            var getterGenerator = getter.GetILGenerator();
            getterGenerator.Emit(OpCodes.Ldarg_0);
            getterGenerator.Emit(OpCodes.Ldfld, field);
            getterGenerator.Emit(OpCodes.Ret);

            var setterGenerator = setter.GetILGenerator();
            setterGenerator.Emit(OpCodes.Ldarg_0);
            setterGenerator.Emit(OpCodes.Ldarg_1);
            setterGenerator.Emit(OpCodes.Stfld, field);
            setterGenerator.Emit(OpCodes.Ret);

            property.SetGetMethod(getter);
            property.SetSetMethod(setter);

            fields.Add(field);
        }
    }

    private void CreateEqualsMethod(TypeBuilder typeBuilder, List<FieldInfo> fields)
    {
        if (typeBuilder == null) throw new ArgumentNullException(nameof(typeBuilder));
        if (fields == null) throw new ArgumentNullException(nameof(fields));

        var method = typeBuilder.DefineMethod("Equals", MethodAttributes.Public | MethodAttributes.ReuseSlot | MethodAttributes.Virtual | MethodAttributes.HideBySig, typeof(bool), new[] { typeof(object) });

        var methodGenerator = method.GetILGenerator();
        var other = methodGenerator.DeclareLocal(typeBuilder);
        var next = methodGenerator.DefineLabel();
        methodGenerator.Emit(OpCodes.Ldarg_1);
        methodGenerator.Emit(OpCodes.Isinst, typeBuilder);
        methodGenerator.Emit(OpCodes.Stloc, other);
        methodGenerator.Emit(OpCodes.Ldloc, other);
        methodGenerator.Emit(OpCodes.Brtrue_S, next);
        methodGenerator.Emit(OpCodes.Ldc_I4_0);
        methodGenerator.Emit(OpCodes.Ret);
        methodGenerator.MarkLabel(next);
        foreach (var field in fields)
        {
            var comparerType = typeof(EqualityComparer<>).MakeGenericType(field.FieldType);
            next = methodGenerator.DefineLabel();
            methodGenerator.EmitCall(OpCodes.Call, comparerType.GetMethod("get_Default"), null);
            methodGenerator.Emit(OpCodes.Ldarg_0);
            methodGenerator.Emit(OpCodes.Ldfld, field);
            methodGenerator.Emit(OpCodes.Ldloc, other);
            methodGenerator.EmitCall(OpCodes.Callvirt, comparerType.GetMethod("Equals", new[] { field.FieldType, field.FieldType }), null);
            methodGenerator.Emit(OpCodes.Brtrue_S, next);
            methodGenerator.Emit(OpCodes.Ldc_I4);
            methodGenerator.Emit(OpCodes.Ret);
            methodGenerator.MarkLabel(next);
        }

        methodGenerator.Emit(OpCodes.Ldc_I4_1);
        methodGenerator.Emit(OpCodes.Ret);
    }

    private void CreateGetHashCodeMethod(TypeBuilder typeBuilder, List<FieldInfo> fields)
    {
        if (typeBuilder == null) throw new ArgumentNullException(nameof(typeBuilder));
        if (fields == null) throw new ArgumentNullException(nameof(fields));

        var method = typeBuilder.DefineMethod("GetHashCode", MethodAttributes.Public | MethodAttributes.ReuseSlot | MethodAttributes.Virtual | MethodAttributes.HideBySig, typeof(int), null);

        var methodGenerator = method.GetILGenerator();
        methodGenerator.Emit(OpCodes.Ldc_I4_0);
        foreach (var field in fields)
        {
            var comparerType = typeof(EqualityComparer<>).MakeGenericType(field.FieldType);
            methodGenerator.EmitCall(OpCodes.Call, comparerType.GetMethod("get_Default"), null);
            methodGenerator.Emit(OpCodes.Ldarg_0);
            methodGenerator.Emit(OpCodes.Ldfld, field);
            methodGenerator.EmitCall(OpCodes.Callvirt, comparerType.GetMethod("GetHashCode", new[] { field.FieldType }), null);
            methodGenerator.Emit(OpCodes.Xor);
        }

        methodGenerator.Emit(OpCodes.Ret);
    }
}
```
