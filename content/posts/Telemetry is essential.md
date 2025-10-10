---
date: 2025-10-10
showTableOfContents: false
tags: ["architecture","software development", "engineering", "programming"]
title: "Telemetry is essential"
type: "post"
image: "/images/telemetry.png"
---
This article won't be about other use cases of telemetry. I do not care about the politics or intentions of companies who utilize telemetry for user tracking. I am interested in telemetry as a tool, and like every tool, it could be used for its intended purpose or to bash someone's head in.

I've been in the outsourcing industry for more than 10 years and while that kind of work lacks in continued purpose, and sometimes, continued involvement it does have its perks. Variety being the biggest one in my opinion. Generalist approach towards software development is my preference and outsourcing industry gives me that opportunity. I've built both frontend(web and desktop) and backend and tools. In all kinds of languages and frameworks - from low level to high level.

One thing that was common across that varied/generalist experience was utter and total disregard towards measuring how the applications are running. I'd wager not even 5% of all blog posts, tutorials, books and videos for building software on the internet had even mentioned of how crucial the importance of telemetry is.

Sooner or later *everyone* will have to fix bugs and fix poor performance in production. It's like an unwritten law about software development. In most cases it turns to guesswork, intuition, some previous experience which vaguely applies to the current situation or just reading log files - if there are any. This is not good enough. Not nearly good enough especially if you call yourself a software **engineer**.
Every single time I had to go down into the mines and fix stuff, tools like [Open Telemetry](https://opentelemetry.io) felt like a Godsend. Tracking a request across the services, measuring latency, memory buildup, function execution time, even pushing memory dumps periodically from tools like [dotnet-gcdump](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-gcdump) made it exceptionally simple to find an issue. Not easy. But simple nonetheless.

Telemetry is essential. It's your choice if you're going to make your life easier down the line when the issues arrise or not. If you're a part of a team and it's your responsibility to architect a solution please telemetry an essential part of it from the start. As the product grows and telemetry has strong roots it will be easier to plug new traces and metrics. It will help you and your team.
