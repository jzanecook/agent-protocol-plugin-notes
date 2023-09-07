# Agent Protocol Spec - WIP

### Contextual Conversational Notes

> Maybe we can create agent protocol extensions and add an endpoint to the agent protocol that lists available extensions.
> 
> @swiftyos, September 4, 2023, 2:16AM (CST)

> I was mentioning an idea of protocol “plugins” to @merwanehamadi which sounds like this a lot
> 
> @mlejva, September 4, 2023, 2:17AM (CST)

> It seems like a good way to keep progress rapid whilst keeping the core protocol lean.
> 
> @swiftyos, September 4, 2023, 2:24AM (CST)

> Yes, we also all hear a lot of developers from the community wanting the protocol to go in different directions. This would allow anyone to build what they want and ultimately developers will decide what will get "accepted" by the field by just using it. Same as with any programming libraries.
We can create a simple nice website with list of all plugins/extensions. 
I think @swyx.io (Shawn)'s proposal for his second protocol might be a good fit. 
> 
> Combined with an Agentfile and great CLI we might make it really easy to adopt with smooth DX. The Agentfile can be something really simple for starters like:
> ```
> version = 1.0.0 # Version of the agent protocol
> plugins = [
>   aief/auth,
>   aief/upload,
>   aief/codegen,
>   aief/logging,
>   some-other-author/agent-to-agent-communication,
> ]
> ```
> 
> Then just run `$ agent-protocol install` or `$ agent-protocol add <package> `
> 
> @mlejva, September 4, 2023, 2:51AM (CST)

---

### Initial Note
I want to avoid confusing an "Agentfile" with a "Dockerfile" or "Makefile", although that analogy doesn't necessarily have to be made, I think it makes more sense in the context of existing package managers

### Questions to Answer
- What would a plugin look like?
    - This would likely look similar to a regular agent, where the Agent.toml file would be responsible for declaring the "agent" components of whatever the codebase is, ultimately pointing towards the API side of the application that holds the agent-protocol connection.
- What would the agent look like with the package code implemented?
- How would you keep the agent-protocol language-agnostic/interoperable with all of these potentially conflicting languages and methods, like binaries vs libraries?

### Notes

For an initial agent-protocol CLI I had been planning on working on one based in Rust, however with the concept of plugins and adding/installing them and managing plugin dependencies it definitely changes the scope a little. For sure, however, I'd like it to be Rust-based CLI for cross-platform and runtime *SPEED*

- Initial idea was to reference Conda: https://docs.conda.io/en/latest/
    > Package, dependency and environment management for any language—Python, R, Ruby, Lua, Scala, Java, JavaScript, C/ C++, Fortran, and more.
    > 
    > Conda is an open source package management system and environment management system that runs on Windows, macOS, and Linux. Conda quickly installs, runs and updates packages and their dependencies. Conda easily creates, saves, loads and switches between environments on your local computer. It was created for Python programs, but it can package and distribute software for any language.

However, making it into a fully fledged package manager would be way too bulky for this scope, since we want to keep it lean. Perhaps the CLI shouldn't be a package manager per se but more so an agent manager (with of course compliance testing tools, etc.).

Where when you run an agent, you would run `agent-protocol start` or something to that effect and the CLI would pull the other agents and run them side by side with the local agent.
- e.g. if you had the `aeif/auth` and `aief/upload` plugins, you would put them into your Agent.toml file and then it would run the commands to set them up and start them.

Calling it the `agent package manager` or `apm` is an option that was presented to me, *this note is ultimately irrelevant*.

So then a potential spec for the Agent.toml file would be as follows:

```yaml
version: '1.0.0'

agent:
    name: My Agent
    workflow:
        - command: yarn && yarn install
    plugins:
        aief/auth:
            version: latest
            features:
                - api_key
                - bearer
        aief/upload: 1.0.2
        some-other-author/agent-to-agent-communication:
            source: git@github.com:some-other-author/agent-to-agent-communication.git
```

**Notice how that's a YAML? On recommendation from @mlejva I've replaced yaml with toml for the following reason:**
> So. Many. Whitespaces.
> 
> *mlejva, September 6, 2023 10:34PM PST*

**I've also refrained from using JSON due to a similar reason:**
> So. Many. Curly braces.
>
> *mlejva, September 6, 2023 10:35PM PST*

**Although I kept the original YAML just for show, in case anybody understands YAML as much as Docker Compose wants you to.**

So, here's a TOML file.
```toml
[protocol]
version = "1.0.0"

[agent]
name = "my-agent"
author = "some-dude"
description = "AI Agent that does stuff"
version = "0.1.0"

[agent.workflow]
setup = "yarn && yarn build"
entrypoint = "yarn start"
# Something I just realized is that just writing "yarn" isn't good enough, because of versions.
# Would we need to require agents to be (pre-built) Docker images to function with this concept?
port = "3000"
# If we did, then we could use whatever ports are available from the Docker image, too...

[plugins]
"aief/auth" = { version = "1.0.0", features = ["api_key", "bearer"] }
"aief/upload" = "1.0.2"
"some-other-author/agent-to-agent-communication" = { source = "git@github.com:some-other-author/agent-to-agent-communication.git" }
```

An example from another agent that would be used as a package would be:
```toml
[protocol]
version = "1.0.0"

[agent]
name = "Agent to Agent Communication"
author = "some-other-author"

[agent.workflow]
setup = "poetry"
entrypoint = "uvicorn app.main:app --host"
port = "3000"

# See note above for why this might just be using a docker image, or not.
```

Current Flow:
```
My Agent
    ↳ Tasks (Create a blackjack game in HTML/CSS and Javascript)
        ↳ Steps:
            - Write Documentation
            - Write HTML File
            - Write CSS File
            - Write JS File
            - Test
        ↳ Artifacts:
            - documentation.txt
            - index.html
            - index.js
            - stylesheet.css

Note: Authentication/Uploads/Etc. would likely have to handled externally, via a wrapper API that then sends data to the agent.
```

Proposed Flow:
```
My Agent
    ↳ Plugin: aief/auth
    ↳ Tasks (Create a blackjack game in HTML/CSS and Javascript)
        ↳ Steps:
            - Ask plugin: agent-to-agent-communication for (e.g.) documentation based on the task input
                ↳ Write Documentation
            - Write HTML File
            - Write CSS File
            - Write JS File
            - Test
        ↳ Artifacts:
            - documentation.txt
            - index.html
            - index.js
            - stylesheet.css
        ↳ Plugin: aief/upload - Task Hook, awaiting Artifact(s)
            ↳ Upload Artifact(s) to Configured Upload Location

Any agents/packages that don't explicitly register as middleware or plugins would run in tandem with this agent.
```

Reference: https://github.com/AI-Engineers-Foundation/agent-protocol/blob/main/client/python/examples/minimal.py

New example:

```python
import asyncio

from agent_protocol.models import StepRequestBody
from agent_protocol_client import (
    Configuration,
    ApiClient,
    StepRequestBody,
    TaskRequestBody,
    AgentApi,
    # I think that this LoadPlugin mechanic could do the trick,
    # but it's a lot of responsibility for one little dependency.
    LoadPlugin,
)

# Defining the host is optional and defaults to http://localhost
# See configuration.py for a list of all supported configuration parameters.
configuration = Configuration(host="http://localhost:8000")


async def main():
    # Enter a context with an instance of the API client
    async with ApiClient(configuration) as api_client:
        # Create an instance of the API class
        api_instance = AgentApi(api_client)

        aief_auth_plugin = LoadPlugin(
            name="aief/auth",
        )
        aief_upload_plugin = LoadPlugin(
            name="aief/upload",
            configuration=aief_upload_configuration
            # Wait a minute, now, how would you pass environment variables to another plugin?
            # I spy a hierarchy!
        )
        agent_to_agent_communication_plugin = LoadPlugin(
            name="some-other-author/agent-to-agent-communication"
            # Now we just need a nice simple way to inject it into *our* agent, and use it effectively.
            # More elaboration needed here.
        )

        task_request_body = TaskRequestBody(input="Write 'Hello world!' to hi.txt.")

        response = await api_instance.create_agent_task(
            task_request_body=task_request_body,
            plugins=[aief_auth_plugin]
        )
        print("The response of AgentApi->create_agent_task:\n")
        print(response)
        print("\n\n")

        task_id = response.task_id
        i = 1

        while (
            step := await api_instance.execute_agent_task_step(
                task_id=task_id, step_request_body=StepRequestBody(input=i)
            )
        ) and step.is_last is False:
            print("The response of AgentApi->execute_agent_task_step:\n")
            print(step)
            print("\n\n")
            i += 1

        print("Agent finished its work!")


asyncio.run(main())
```

---

Notes for tomorrow:
- Spend the night sleeping
- Use the added creativity from the morning to revise and create a proper draft of a spec
- Elaborate on the concept of the CLI and make it more fleshed out in general

Also, keep in mind the following quote:
> 1. I don’t think a protocol or a plugin should be an agent. That’s very very specific implementation. Protocols should be just definitions and the implementation can be whatever (read oauth rfc for example)
> 2. I would not worry about agentfile. Instead I would focus on how plugin system works
> 3. Make it simple in the beginning, don’t assume many things. Instead get it out fast and iterate from the real life feedback
> 4. Please don’t use yaml 😄
> 
> *mlejva, September 6 2023 10:26PM PST*

And figure out exactly what it means and what you could cut out or re-prioritize for a future update. Or, whether you've been going in the wrong direction the whole time.