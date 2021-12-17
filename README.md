
All code from this tutorial as a complete package is available in [this repository](https://github.com/alexeagleson/monorepo-example).

The purpose of this tutorial is to learn about some of the different ways that you can structure a large project that is built from multiple smaller projects.

### Monorepos

One method of grouping code from multiple projects into one is called a [monorepo](https://en.wikipedia.org/wiki/Monorepo).  A _monorepo_ is simply the practice of placing multiple different projects that are related in some way into the same repository.

The biggest benefit is that you do not need to worry about version mismatch issues between the different pieces of your project.  If you update an API route in the server of your monorepo, that commit will be associated with the version of the front end that consumes it.  With two different repositories you could find yourself in a situation where your v1.2 front-end is asking for data from your v1.1 backend that somebody forgot to push the latest update for.

Another big benefit is the ability to import and share code and modules between projects.  Sharing types between the back-end and front-end is a common use case.  Your can define the shape of the data on your server and have the front-end consume it in a typesafe way.  

### Git Submodules

In addition to monorepos, we also have the concept of [submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules).

Let's say that we want to add a feature to our app that we have in another separate project.  We don't want to move the entire project into our monorepo because it remains useful as its own independent project.  Other developers will continue  to work on it outside of our monorepo project.

We would like a way to include that project inside our monorepo, but not create a separate copy.  Simply have the ability to pull the most recent changes from the original repository, or even make our own contributions to it from inside our monorepo.  Git submodules allows you to do exactly that.

This tutorial will teach you how to create your own project that implements both of these features.

## Table of Contents

1. [Prerequisites and Setup](#prerequisites-and-setup)
1. [Initializing the Project](#initializing-the-project)
1. [Create the React App](#create-the-react-app)
1. [Create the Monorepo](#create-the-monorepo)
1. [Create Your Repository](#create-y0our-repository)
1. [Sharing Code and Adding Dependencies](#sharing-code-and-adding-dependencies)
1. [Add a Git Submodule](#add-a-git-submodule)
1. [Wrapping Up](#wrapping-up)

## Prerequisites and Setup

This tutorial assumes you have a basic familiarity with the following.  Beginner level experience is fine for most as the code can be simply copy/pasted.  For git you should know how to clone, pull, commit and push.

- Git
- React
- Node.js
- Typescript
- NPM

This tutorial requires yarn v1 installed (we use v1.22).  

## Initializing the Project

To start, we need a `packages` directory to hold the different projects in our monorepo.  Your structure should begin looking like this:


```
.
└── packages
    └── simple-express-app
          └── server.ts
      
From within the `packages/simple-express-app` directory, run:

```bash
yarn init

yarn add express

yarn add -D typescript @types/express

npx tsc --init
```

The final command will create a `tsconfig.json` file.  Add the following to it:

`packages/simple-express-server/tsconfig.json`
```json
{
  ...
  "outDir": "./dist",
}
```

Now create your server file if you haven't yet:

`packages/simple-express-server/server.ts`

```ts
const express = require("express");
const app = express();
const port = 3001;

app.get("/data", (req, res) => {
  res.json({ foo: "bar" });
});

app.listen(port, () => {
  console.log(`Example app listening at http://localhost:${port}`);
});
```

At this point your directory structure should look like:

```
.
└── packages
    └── simple-express-app
          ├── server.ts
          ├── yarn.lock
          ├── package.json
          └── tsconfig.json
```

We'll create a simple script in `package.json` called `start` that we can run with `yarn`:

`packages/simple-express-server/package.json`
```json
{
  "name": "simple-express-server",
  "version": "1.0.0",
  "main": "dist/server.js",
  "license": "MIT",
  "scripts": {
    "start": "tsc && node dist/server.js"
  },
  "devDependencies": {
    "@types/express": "^4.17.13",
    "typescript": "^4.5.4"
  },
  "dependencies": {
    "express": "^4.17.1"
  }
}
```

Open your browser to [](http://localhost:3001/) and you will see your data successfully queried:

![Express Data](https://res.cloudinary.com/dqse2txyi/image/upload/v1639709686/blogs/git-submodules/express-data_i45ghb.png)

## Create the React App

Next we move onto our React app.  Navigate to the `packages` directory and run this command:

```bash
yarn create react-app simple-react-app --template typescript
```

Before we do anything else we want to confirm that we can communicate with our server and get the JSON data that we are serving up.

Open up the `App.tsx` file in the `src` directory of the project generated by `create-react-app`.  We are going to add a simple button that uses the browser [fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) to grab the data from our server and log it to the console.  

`packages/simple-react-app/src/App.tsx`
```tsx
import React from "react";
import logo from "./logo.svg";
import "./App.css";

function App() {
  return (
    <div className="App">
      <header className="App-header">
        <img src={logo} className="App-logo" alt="logo" />
        <p>
          Edit <code>src/App.tsx</code> and save to reload.
        </p>
        <a
          className="App-link"
          href="https://reactjs.org"
          target="_blank"
          rel="noopener noreferrer"
        >
          Learn React
        </a>
        
        { /* NEW */ }
        <button
          onClick={() => {
            fetch("http://localhost:3001/", {})
              .then((response) => response.json())
              .then((data) => console.log(data));
          }}
        >
          GET SOME DATA
        </button>

      </header>
    </div>
  );
}

export default App;
```

When we open the browser's development console (F12) and then click our button, we will see our server data fetched and logged in the browser:

![React Fetch Example](https://res.cloudinary.com/dqse2txyi/image/upload/v1639711081/blogs/git-submodules/react-fetch_x3iqd8.png)

This is great!  We've accidentally created a template for a full stack React and Typescript app!  But that's not the reason we're here, so let's start pushing further into scenarios we might encounter in real projects that would lead us to consider options like a monorepo and git submodules.

Before you continue take a moment to verify your project structure:

```
.
└── packages
    ├── simple-express-server
    │   ├── server.ts
    │   ├── yarn.lock
    │   ├── package.json
    │   └── tsconfig.json
    └── simple-react-app
        └── [default setup]
```

## Create the Monorepo

To manage our monorepo we are going to use two tools:

* [Lerna](https://lerna.js.org/): For running scripts across multiple projects and adding new dependencies.  Lerna is also built to manage publishing your packages (though we will not be doing that as part of this tutorial)

* [Yarn workspaces](https://yarnpkg.com/features/workspaces): For hoisting all shared dependencies into a single `node_modules` folder in the root directory.  Each project can still define its own dependencies, so that you don't confuse which dependencies are required for which (client vs. server) for example, but it will pool the installed packages in the root.  

For yarn we are using the still most commonly used yarn v1 _(current version as of this writing is v1.22)._

Navigate to the root directory and run the following commands:

```bash
yarn init

yarn add -D lerna typescript

npx lerna init
```

Edit your Lerna configuration file:

```json
{
  "packages": ["packages/*"],
  "version": "0.0.0",
  "npmClient": "yarn",
  "useWorkspaces": true
}
```

We need to specify that `yarn` is our NPM client and that we are using workspaces.

Next we need to define the location of those workspaces in the root `package.json`:

`package.json`
```json
{
  "name": "monorepo-example",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "private": true,
  "workspaces": [
    "packages/*"
  ],
  "scripts": {
    "start": "lerna run --parallel start"
  },
  "devDependencies": {
    "lerna": "^4.0.0"
  }
}
```

We have made three changes above:

* Set `private` to `true` which is necessary for workspaces to functions

* Defined the location of the workspaces as `packages/*` which matches any directory we place in `packages`

* Added a script that uses Lerna to run.  This will allow us to use a single command to run the equivalent of `yarn start` in both our Express server and React app simultaneously.  This way they are coupled together so that we don't accidentally forget to run one, knowing that currently they both rely on each other.  The `--parallel` flag allows them to run at the same time.

Now we are ready to install the dependencies in root:

_(Note: At this point before you run the install command, I would recommend you synchronize your Typescript version between your `simple-express-server` and the one that comes bundled with your `simple-react-app`.  Make sure both versions are the same in each project's `package.json` and both are listed in `devDependencies`.  Most likely the React app version will be older, so that is the one that should be changed.)_

Next run the following command:

```bash
npx lerna clean -y

yarn install
```

The first command will clean up the old `node_modules` folders in each of your two packages.  This is the equivalent of simply deleting them yourself.  

The second command will install all dependencies for both projects in a `node_modules` folder in the root directory.

Go ahead and check it out!  You'll see that `node_modules` in the root is full of packages, while the `node_modules` folders in `simple-express-server` and `simple-react-app` only have a couple (these are mostly symlinks to binaries that are necessary due to the way yarn/npm function).

Before we go on we should create a `.gitignore` file in the root to make sure we don't commit our auto-generated files:

`.gitignore`
```
node_modules/
dist/
```

_(If you're using VS Code you'll see the folder names in the side bar go grey as soon as you sae the file, so you know it worked)_

Verify your monorepo and workspaces are setup properly by running (from the root folder):

```
yarn start
```
You will see that both your Express app and React app start up at the same time!  Click the button to verify that your server data is available and logs to the console.  

Lastly we need to initialize Typescript in the root of the project so that our different packages can import and export between one another.  Run the command:

```bash
npx tsc --init
```

In the root directory and it will create your `.tsconfig.json`.  You can delete all the defaults values from this file (your individual projects will se their own configuration values.)  The only field you need to include is:

`tsconfig.json`
```json
{
  "compilerOptions": {
    "baseUrl": "./packages"
  }
}
```

Our project now looks like:

```
.
├── packages
|   ├── simple-express-server
|   │   ├── server.ts
|   │   ├── yarn.lock
|   │   ├── package.json
|   │   └── tsconfig.json
|   └── simple-react-app
|       └── [default setup]
├── lerna.json
├── tsconfig.json
├── package.json
└── yarn.lock
```

## Create Your Repository

This is also a good time to commit your new project to your repository.  I'll be doing that now as well, you can see the [final version here](https://github.com/alexeagleson/monorepo-example).

Note that in order to learn submodules effectively, we are going to be adding a submodule from a repository that _already exists_, we don't want to use the one that `create-react-app` generated automatically.  

**So for that reason I am going to delete the that repository by deleting the `.git` directory inside `packages/simple-react-app`.  This step is VERY IMPORTANT.  Make sure there is no `.git` directory inside `simple-react-app`.**

Now from the root directory you can run:

```bash
git add .
git commit -am 'first commit'
git remote add origin YOUR_GIT_REPO_ADDRESS
git push -u origin YOUR_BRANCH_NAME
```

## Sharing Code and Adding Dependencies

So let's quickly take a look at some of the benefits we get from our monorepo.  

Let's say that there's a utility library that we want to use in both our React app and on our Express server.  For simplicity let's choose [lodash](https://lodash.com/) which many people are familiar with.

Rather than adding it to each project individually, we can use `lerna` to install it to both.  This will help us make sure that we keep the same version in sync and require us to only have one copy of it in the root directory.  

From the root run the following command:

```bash
npx lerna add lodash packages/simple-*

npx lerna add @types/lodash packages/simple-* --dev
```

This will install `lodash` in any of the projects in the `packages` directory that match the `simple-*` pattern (which includes both of ours).  When using this command you can install the package to dev and peer dependencies by adding `--dev` or `--peer` at the end.  More info on this command [here](https://github.com/lerna/lerna/tree/main/commands/add#lernaadd).

If you check the `package.json` file in both your packages you'll see that `lodash` has been added with the same version to both files, but the actual package itself has a single copy in the `node_modules` folder of your root directory.  

So we'll update our `server.ts` file in our Express project to do a couple of new things.  We'll import the shared `lodash` library and use one of its functions (`_.snakeCase()`) and we'll define a type interface that defines the shape of the data we are sending and export it so that we can _also_ use that interface in our React app to typesafe server queries.  

Update your `server.ts` file to look like the following:

`packages/simple-express-server.ts`
```ts
import express from "express";
import _ from "lodash";
const app = express();
const port = 3001;

export interface QueryPayload {
  payload: string;
}

app.use((_req, res, next) => {
  // Allow any website to connect
  res.setHeader("Access-Control-Allow-Origin", "*");

  // Continue to next middleware
  next();
});

app.get("/", (_req, res) => {
  const responseData: QueryPayload = {
    payload: _.snakeCase("Server data returned successfully"),
  };

  res.json(responseData);
});

app.listen(port, () => {
  console.log(`Example app listening at http://localhost:${port}`);
});
```

_(Note I have changed the key on the object from `data` to `payload` for clarity)_

Next we will update our `App.tsx` component in `simple-react-app`.  We'll import `lodash` just for no other reason to show that we can import the same package in both client and server.  We'll use it to apply `_.toUpper()` to the "Learn React" text.  

We will also import our `QueryPayload` interface from our `simple-express-server` project.  This is all possible through the magic of workspaces and Typescript.


`packages/simple-react-app/src/App.tsx`
```tsx
import React from "react";
import logo from "./logo.svg";
import "./App.css";
import _ from "lodash";
import { QueryPayload } from "simple-express-server/server";

function App() {
  return (
    <div className="App">
      <header className="App-header">
        <img src={logo} className="App-logo" alt="logo" />
        <p>
          Edit <code>src/App.tsx</code> and save to reload.
        </p>
        <a
          className="App-link"
          href="https://reactjs.org"
          target="_blank"
          rel="noopener noreferrer"
        >
          {_.toUpper("Learn React")}
        </a>
        <button
          onClick={() => {
            fetch("http://localhost:3001/", {})
              .then((response) => response.json())
              .then((data: QueryPayload) => console.log(data.payload));
          }}
        >
          GET SOME DATA
        </button>
      </header>
    </div>
  );
}

export default App;
```

I find this is one of the trickiest parts to get right (the importing between packages).  The key to this is the installation of Typescript in the root of the project, and `"baseUrl": "./packages"` value in the the `tsconfig.json` in the root directory.  

If you continue to have difficulty [this](https://stackoverflow.com/a/61467483) is one of the best explanations I have ever come across for sharing Typescript data between projects in a monorepo.

Once everything is setup, press the button on your React application and you'll be greeted with:

![React Monorepo Fetch Example](https://res.cloudinary.com/dqse2txyi/image/upload/v1639718441/blogs/git-submodules/react-monorepo-fetch_p5hoys.png)

Notice the snake_case response that matches the correct shape we defined.  Fantastic. 

Let's look at git submodules.  

## Add a Git Submodule

Recently I wrote a blog post on a very simple component for a React app that adds a dark mode, a `<DarkMode />` component.  The component is not part of a separate library we can install with an NPM command, it exists as part of a React application that has its own repository.

Let's add it to our project, while still keeping it as its own separated repo that can be updated and managed independent of our monorepo.  

From the `packages/simple-react-app/src` directory we'll run this command:

```bash
git submodule add git@github.com:alexeagleson/react-dark-mode.git
```

That will create the `react-dark-mode` directory (the name of the git repository, you can add another argument after the above command to name the directory yourself).

To import from the submodule it's as simple as... importing from the directory.  If we're going to add the `<DarkMode />` component it's as simple as adding:

`packages/simple-react-app/src/App.tsx`
```tsx
...
import DarkMode from "./react-dark-mode/src/DarkMode";

function App() {
  return (
    <div className="App">
      ...
      <DarkMode />
    </div>
  );
}

export default App;
```

I've omitted some of the repetitive stuff above.  Unfortunately the default `background-color` styles in `App.css` are going to override the `body` styles, so we need to update `App.css` for it to work:

`packages/simple-react-app/src/App.css`
```css
...

.App-header {
  /* background-color: #282c34; */
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  font-size: calc(10px + 2vmin);
  /* color: white; */
}

.App-link {
  /* color: #61dafb; */
}

...
```

Comment out those color values and you're good to go!

![React Submodule Import Example](https://res.cloudinary.com/dqse2txyi/image/upload/v1639720663/blogs/git-submodules/react-submodule-import_cmdsk3.png)

Now you might be thinking -- couldn't I just have cloned that repo into that folder and done this?  What's the difference with submodules?

Well now that we have this in place, let's look for the answer to exactly that.  Run the following command:

```bash
git status
```

In the output you'll see `new file:   ../../../.gitmodules`.  That's something new if you've never used submodules before.  It's a hidden file that has been added to the project root.  Let's take a look inside it:

```gitmodules
[submodule "packages/simple-react-app/src/react-dark-mode"]
	path = packages/simple-react-app/src/react-dark-mode
	url = git@github.com:alexeagleson/react-dark-mode.git
```

It stores a mapping to the directories in our project that map to other repositories.  

Now if you commit your changes in the root of the monorepo and push, you'll see on Github that rather than being a regular directory inside this project -- it's actually a link to the real repository:

![Github Submodules](https://res.cloudinary.com/dqse2txyi/image/upload/v1639722122/blogs/git-submodules/github-submodule_tel7cc.png)

So you can continue to update and make changes to this monorepo without impacting that other repository.  Great!  

But can you update the dark mode repository from inside this one?  Sure you can!  (As long as you have write permission).

Let's make a trivial change to the dark mode repository from inside this one and see what happens.  Navigate to:

`packages/simple-react-app/src/react-dark-mode/src/DarkMode.css`
```css
...
[data-theme="dark"] {
  --font-color: #eee;
  --background-color: #333;
  --link-color: peachpuff;
}
```

I'm going to update the colour of the link when the app is in dark mode, from `lightblue` to `peachpuff`.

Now obviously you won't be able to update my repository, but if you're following you can continue reading to see where this is going (or you can use your own repository of course).

From this directory I make a commit and push.  When I check the repository there are no new commits to the `monorepo-example` repository, but there IS a new commit to `react-dark-mode`.  Even though we are still inside our monorepo project!

![Github Submodule Commit Example](https://res.cloudinary.com/dqse2txyi/image/upload/v1639722643/blogs/git-submodules/github-submodule-2_jyb55y.png)

When working with submodules it's important to keep them up to date.  Remember that other contributors could be making new commits to the submodules.  The regular `git pull` and `git fetch` to your main root monorepo aren't going to automatically pull new changes to submodules.  To do that you need to run:

```bash
git submodule update
```

To get the latest updates.  

You also have new command you'll need to run when cloning a project or pulling when new submodules have been added.  When you use `git pull` it will pull the information _about_ relevant submodules, but it won't actually pull the code from them into your repository.  You need to run:

```
git submodule init
```

To pull the code for submodules.

Lastly, in case you prefer not to run separate commands, there is a way to pull submodule updates with your regular commands you're already using like clone and pull.  Simply add the `--recurse-submodules` flag like so:

```
git pull --recurse-submodules

or

git clone --recurse-submodules
```

## Wrapping Up

I hope you learned something useful about monorepos and submodules.  THere are tons of different ways to setup a new project, and there's no one-size-fits-all answer for every team.

I'd encourage you to play around with small monorepos (even clone this example) and get get comfortable with the different commands.  


Please check some of my other learning tutorials.  Feel free to leave a comment or question and share with others if you find any of them helpful:

- [Learnings from React Conf 2021](https://dev.to/alexeagleson/learnings-from-react-conf-2021-17lg)

- [How to Create a Dark Mode Component in React](https://dev.to/alexeagleson/how-to-create-a-dark-mode-component-in-react-3ibg)

- [How to Analyze and Improve your 'Create React App' Production Build ](https://dev.to/alexeagleson/how-to-analyze-and-improve-your-create-react-app-production-build-4f34)

- [How to Create and Publish a React Component Library](https://dev.to/alexeagleson/how-to-create-and-publish-a-react-component-library-2oe)

- [How to use IndexedDB to Store Local Data for your Web App ](https://dev.to/alexeagleson/how-to-use-indexeddb-to-store-data-for-your-web-application-in-the-browser-1o90)

- [Running a Local Web Server](https://dev.to/alexeagleson/understanding-the-modern-web-stack-running-a-local-web-server-4d8g)

- [ESLint](https://dev.to/alexeagleson/understanding-the-modern-web-stack-linters-eslint-59pm)

- [Prettier](https://dev.to/alexeagleson/understanding-the-modern-web-stack-prettier-214j)

- [Babel](https://dev.to/alexeagleson/building-a-modern-web-stack-babel-3hfp)

- [React & JSX](https://dev.to/alexeagleson/understanding-the-modern-web-stack-react-with-and-without-jsx-31c7)

- [Webpack: The Basics](https://dev.to/alexeagleson/understanding-the-modern-web-stack-webpack-part-1-2mn1)

- [Webpack: Loaders, Optimizations & Bundle Analysis](https://dev.to/alexeagleson/understanding-the-modern-web-stack-webpack-part-2-49bj)

---

For more tutorials like this, follow me <a href="https://twitter.com/eagleson_alex?ref_src=twsrc%5Etfw" class="twitter-follow-button" data-show-count="false">@eagleson_alex</a><script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> on Twitter