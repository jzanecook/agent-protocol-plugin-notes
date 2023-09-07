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

**This implementation is likely flawed**
- Adding a route to register external agents or plugins to this agent makes more sense. The implementation of the execute_agent_task_step or the create_agent_task would need to keep in mind the initial registration of outside sources. That's *if* this concept is valid.

---

Also, keep in mind the following quote:
> 1. I don’t think a protocol or a plugin should be an agent. That’s very very specific implementation. Protocols should be just definitions and the implementation can be whatever (read oauth rfc for example)
> 2. I would not worry about agentfile. Instead I would focus on how plugin system works
> 3. Make it simple in the beginning, don’t assume many things. Instead get it out fast and iterate from the real life feedback
> 4. Please don’t use yaml 😄
> 
> *mlejva, September 6 2023 10:26PM PST*

My interpretation of this message is to focus on more specification-focused pursuits. With this in mind, the implementation of the agent-protocol library should be less my focus, and I should attempt to focus instead on creating a "standard". Like the RFC he mentioned.

The issue I have with the concept of a "protocol or plugin shouldn't be an agent" is that, then, well, what *should* a plugin look like? How would it be introduced to the agent?

So with that in mind, a quick overview of what the agent-protocol CLI would look like or function as in this setup just so I don't have to think about it again:
- `$ agent-protocol init` - Set up a new Agent
    - Flags:
        - `--template, -t` - Specify a github based template as a base for the new agent. (e.g. minimal)
        - A set of flags would be presented that would automate any questions asked during the normal process, along with an option to skip them entirely like `--yes, -Y`
    - Arguments:
        - `path` - (Default: `./`) The path of the new Agent, would automatically be the name.
    - Process: Prompt the user with a set of questions, like the author name, project name, description, and any other necessary metadata and create the Agent.toml file at a minimum, or if using a template, pull the template using Git into the directory.
- `$ agent-protocol info` - Display a formatted infodump of the Agent.toml file
- `$ agent-protocol add <package>` - Verifies the package is available (can be an `author/name` or a github repo with an Agent.toml file in the root of whatever folder is specified)
- `$ agent-protocol remove <package>` - Self explanatory
- `$ agent-protocol start` - Starts up the current agent, and all associated plugins/agents/packages.

I think this would serve as a solid base for the agent-protocol CLI tool, however there may be differing opinions on what it would or should exactly entail, that I look forward to hearing.

---

The agent-protocol itself, and the broad specification of an Agent Protocol, would include a modified version of the existing protocol as mentioned above.

An agent consists of an API (with a variable URL/Port/SSL strategy) where it consists of several standardized routes, in reference to the openapi spec below.

```
/agent/tasks - POST, GET - Creates a task for the agent or gets all tasks previously created for the agent.
/agent/tasks/{task_id} - GET - Gets details about a specific task.
/agent/tasks/{task_id}/steps - GET, POST - Lists all steps for the specified task, or initiates a step for the specified task.
/agent/tasks/{task_id}/steps/{step_id} - GET - Gets details about a specified task step.
/agent/tasks/{task_id}/artifacts - GET, POST - Gets the list of artifacts for the specified task, or uploads an artifact to the agent for that task.
/agent/tasks/{task_id}/artifacts/{artifact_id} - GET - Downloads a specified artifact.
```

In addition to these, an agent (that could become a plugin) would need to register tasks/steps to be used as plugins.

So, perhaps:
```
/agent/registered - POST, GET - Registers a task as a "registered" task with steps or with artifacts. Ideally this task should be able to be called again in the future, via another agent to receive an output.
```
*This would imply the following:*
```
/agent/registered/{task_id}/
/agent/registered/{task_id}/steps/{step_id}
/agent/registered/{task_id}/artifacts
/agent/registered/{task_id}/artifacts/{artifact_id}
```

**Alternatively:**
Rather than registering tasks/steps you could register the agent itself (that may be necessary).

e.g.:
```
/agent/registered/{agent_author}/{agent-name}/tasks
```

With this approach, the schema/metadata for tasks would be where things like middleware or tools would be implemented. So a task would be like:
```json
{
    "task_id": "b225e278-8b4c-4f99-a696-8facf19f0e56",
    "artifacts": [],
    "plugin": { // Added if the task needs to be run in tandem with other tasks
        "run_at": "start", // start, end
    },
    "tool": { // Added if the task is a tool to be used by another task
        "name": "Upload to S3",
        "requires": ["artifacts", "task_id", "step_id"]
    },
    "blocking": true, // Added if the task needs to stop other tasks/steps from executing.
}
```

Tasks like these would need to be registered rather than triggered, so (within the agent that contains the plugin/tool) you would register the task. Or, perhaps a simple `"registered": true` flag could be added to the task information, which would be available via another agent when you GET `/agent/registered/{author}/{agent_name}/tasks`

More discussion necessary.

### Modifications to Task/Step/Artifact Schema

Some modifications may need to be made to the specification for tasks, steps and artifacts, for instance:
- For steps/tasks: Steps would need to include a blocking strategy for next steps, and there would need to be an implementation to make them run before other tasks/steps in a series, or after. Additionally, they would also need to be called via other steps in the event of using another agent as a plugin, or using a tool, so steps or tasks need to have some form of labelling mechanic introduced in the registration to give the name, description, and arguments that are necessary for this functional approach.
- For artifacts: In the case of something like an auth plugin, being able to freely view the artifacts that it has or produces could become a security issue, adding the ability to make them "private" so that only the agent they're contained in may be necessary.

### Additional Note about API routes for interoperability
If you wanted to build a tool or plugin, you would only really need the routes related to that. So the protocol could be modified to have optional routes that aren't necessary in the grand scheme of things.

e.g. a plugin that includes a task and step but no artifacts wouldn't necessarily need to include the artifacts in the response, or an artifacts API route.