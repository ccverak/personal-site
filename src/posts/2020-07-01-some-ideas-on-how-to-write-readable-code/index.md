---
draft: false
title: "Some ideas on how to write readable code"
date: "2020-07-11"
description: "This is how I use conceptual compression to better express my intent and make my code more readable and understandable"
tags: general
keywords: "clean code, ruby, javascript"
---

Across the years in my career as a software developer, I have always felt that I should get better at making my code more readable. This keeps coming to me every time I read my code from the past and find it hard to understand. What was I thinking when I wrote this piece of code? What did I want to communicate and how is it connected to the problem and to the domain? If this happens to me with my own code, what should I expect from others? At some point I decided to take action and make improvements in my coding style for good, hoping to lower the effort from readers -including myself- when understanding my code.

DISCLAIMER: Bear in mind that this is my current mental state about the subject but this is something that I'm sure it will evolve with time. Take this as an intent to spark your interest in the topic and also as some sort of feedback request to readers.

## COMMUNICATING INTENT

In my opinion, writing programs should be more about communicating your intent, not only to the computer but to your fellow developers who are going to read the code at some point. Computers are very smart understanding what you tell them to do, they don't care if the variables have descriptive names or if you used this many files to solve the problem. But guess what? Humans do care a lot about that. A lot. People come and go but the code stays. That is why I think you have to make it really easy for the reader to understand your code. But, how can the code be more understandable? Let me give it a shot:

What makes a code fragment more understandable?

- Knowledge of programming and its essentials
- Knowledge of the programming language
- Understanding of the business domain
- Understanding of the problem
- How the code transmits its intent

After the basics that should be taken for granted such as programming essentials and programming language knowledge, business domain and the problem take special importance. As you may know it is not only about using good variable names, you should ask yourself how your code expresses the connections between the business domain and the solution of the problem, so it's easier to understand by others.

Here is Conceptual Compression can help, let's see how.

## CONCEPTUAL COMPRESSION WHAT?

**Conceptual compression** is all about **abstraction**. It's a mapping of multiple different pieces related to a concept into a single piece. Take for example this definition _Gardening is the practice of growing and cultivating plants as part of horticulture_. Now you know what Gardening means, I can use it in a conversation without explaining its meaning.

## COMPRESSING CONCEPTS IN SOFTWARE DEVELOPMENT

We, the software developers, are prone to create complexity on the top of the existing problem complexity, this tends to accumulate with the time waiting to bite you in the worst moment ever. This is why I think that Conceptual Compression can be a good tool to help you sail through the complexity and time. It will force you to structure your concepts allowing you to create programs composing them without exposing details that aren't important. This helps also to minimize the interference of the accidental complexity created by developers.

```javascript
// Top-level orchestration
function program () {
  doThis()
  doThisOtherThing()
}

// elsewhere

function doThis () {
  /* details implementation */
}

// somewhere else

function doThisOtherThing () {
  /* details implementation */
}
```

### LET'S TALK ABOUT CODE

First and foremost, I'm talking about application code, not about library code. Bear this in mind from now own, this may or may not apply to all the contexts.

**Getting started**

How we can communicate our intent and make use of Conceptual Compressions?

Let's see some example with different language constructs:

### With variable names

Given this piece of code

```javascript
const u = getCurrentUser()

// ...

if (
  project.ownerId === req.session.userId ||
  userPermission.name === 'MANAGE_PROJECTS'
) {
  // ...
}
```

What if we instead implement it like this?

```javascript
const currentUser = getCurrentUser()

// ...

const hasPermission = userPermission.name === 'MANAGE_PROJECTS'
const isProjectOwner = project.ownerId === req.session.userId

if (hasPermission || isProjectOwner) {
  // ...
}
```

Isn't the second example much better? isn't the `if` statement much understandable? The main reason is that it uses concepts that encapsulates its meaning in variable names. This becomes more useful as the number or complexity of the concepts increases.

### With functions (or classes)

```javascript
app.put('/project/:projectId', async ({ req, res }) => {
  const project = await Projects.find(req.projectId)
  if (!project) throw ProjectNotFoundError()
  const userPermission = await ProjectPermissions.find({
    currentUserId,
    userId
  })
  if (
    project.ownerId !== req.session.userId ||
    userPermission.name !== 'MANAGE_PROJECTS'
  ) {
    throw new ForbiddenError()
  }
  const pr = await Projects.update({ projectId, params: req.params })
  return res.json(pr)
})
```

Using concept compression now:

```javascript
function canUpdateProject ({ project, requesterId }) {
  const isOwner = project.ownerId === requesterId

  return isOwner
    ? true
    : hasUpdatePermission({ id: project.id, userId: requesterId })
}

//----------------

async function ensurePermissions ({ project, requesterId }) {
  const canUpdateProject = await canUpdateProject({ project, requesterId })

  if (!canUpdateProject) throw new AuthorizationError()
}

//----------------

async function updateProject (id, requesterId, params) {
  const project = Projects.find(id)

  await ensurePermissions({ project, requesterId })

  return Projects.update(id, params)
}

//-----------------

app.put('/project/:projectId', async ({ request, response }) => {
  const { projectId } = request.params
  const { name, description, category } = request.projectParams

  const currentUserId = currentUserId(request)

  const updatedProject = await updateProject(projectId, currentUserId, {
    name,
    description,
    category
  })

  const presentedProject = presentProject(updatedProject)
  response.json(presentedProject)
})
```

Notice that the proposed implementation is longer and probably would involve more files, but it is better structured, several concepts were extracted such as `canUpdateProject`, `updateProject`, `ensurePermissions` or `presentProject`. The main idea is that if you want to understand the `app.put` logic you don't see irrelevant details.

### With modules

Modules provide a way to encapsulate common functionality exposing what is done while abstracting how is done.

Defining clearer interfaces and keeping internal details separated from the callers have many benefits. It lowers the cognitive load required to understand and use the functionality, it provides a compressed concept that can be orchestrated in a higher level of abstraction without polluting it with details. From the insider point of view, you have more freedom to change the implementation details of the module without risking too much. Also, since the public interface is clearly defined, you will know better when you introducing a breaking change. It helps a lot to construct a well-structured application if you want more details [read this other article of mine](/posts/aiming-for-simplicity/).

Let's see an example with the same `updateProject` endpoint:

```javascript
//----------------- src/projects/commands/index.js
import ensurePermissions from 'not-relevant-now'
import Projects from 'not-relevant-now'

const updateProject = async function updateProject (id, requesterId, params) {
  const project = Projects.find(id)

  await ensurePermissions({ project, requesterId })

  return Projects.update(id, params)
}

export default {
  updateProject
}

//----------------- src/projects/web/index.js

import commands from 'src/projects/commands'
import presentRole from 'not-relevant-now'
import currentUserId from 'not-relevant-now'

app.put('/project/:projectId', async ({ request, response }) => {
  const { projectId } = request.params
  const { name, description, category } = request.projectParams

  const currentUserId = currentUserId(request)

  const updatedProject = await commands.updateProject(
    projectId,
    currentUserId,
    {
      name,
      description,
      category
    }
  )

  const presentedProject = presentProject(updatedProject)
  response.json(presentedProject)
})
```

Notice that the `updateProject` action was compressed and moved into a `commands` module which encapsulates the actions in a way that the external dependencies of this module don't need to be aware of its details. The `app.put` action it's more cleaner, it's easier to understand, and easier to debug. When an error occurs you will be taken to the required context following the stack-trace not having unnecessary details.

**Things to notice**

- It's longer and it requires more files
- Web related actions (`src/projects/web/index.js`) are separated from application logic (`commands.updateProject`)
- The interactions and components are more explicit
- The module `commands` exposes its public API including a way to update projects (it can include more commands, probably separated into different files)
- It communicates the intent in a better way
- It is (hopefully) easier to follow

**When you should use Conceptual Compression**

I'm a web developer and primarily I had had web applications in mind while writing this article. Web applications are more about expressing business rules and interaction between different concepts.

I'm totally aware that for example algorithms, protocols, Systems Programming, Critical Systems, etc. are in a different game where abstractions play a point against performance, I would say that in those cases as long as you empathize enough with the reader to meet the requirements it is more than enough.

Having said that I think that given the context of the problem you are trying to solve, you need to answer these basic questions to know what path to take

A) Do you need the implementation details in the surface?

B) Do you need the interactions in the surface?

## CONCLUSION

I hope I caught your attention in the topic and gave you some food for thought. Here are some takeaways

Concept Compression in application development:

- Uses business wording for concepts
- Helps you to communicate your intent through code
- Provides a path to create better interactions in your code

Business domain problems are responsible of the hardest problems and bugs I've seen, the real-life is hard to model and that makes difficult not to make mistakes in the interactions not only in the details. I don't know if you agree but the hardest problems I've seen are related to the interaction and flow of programs solving business problems. Don't you think that Conceptual compression takes a step in the right direction to make better and reliable interactions?

Till the next time :}

PS: I would like to share also some Ruby code which conceptual compression, it's just a sample class that clones a project from a git repository and then deploys it and then a module that exposes it to others parts of the application.

```ruby
class Projects::Deploy

  # follows the callable interface of blocks

  def self.call(*args)
    new(*args).call
  end

  def initialize(project)
    @project = project
  end

  def call
    return unless deployable?

    clone

    deploy
  end

  private

  def git_repository
    project.git_repository
  end

  def deployment_config
    project.deployment_config
  end

  def clone 
    # clones using git_repository
  end

  def deploy 
    # deploys using deployment_config
  end

  def deployable? 
    # ...
  end

  attr_reader :project
end

# module and public interface

module Projects

  def deploy(project)
    Deploy.call(project)
  end

end
```
