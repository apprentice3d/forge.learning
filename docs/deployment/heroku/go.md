# Heroku 

First, create and activate your [Heroku account](https://www.heroku.com/).

## Prerequisites

Heroku manages app deployments with Git, the popular version control system. The Heroku Command Line Interface (CLI) makes it easy to create and manage your Heroku apps directly from the terminal. It’s an essential part of using Heroku.

- [Install Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- [Install Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli)

## Prepare your project

### Dependency setup
For Go projects, Heroku expects your project to use a vendoring tool (a tool that "holds track" of the version of the dependencies used in your project).
One of such tool is [dep](), that could be install by opening the terminal (menu **View** >> **Integrated terminal**) and typeing:

```bash
    go get dep
```

After that, to have to run specify the dependencies used in your project by typing in the terminal, at root of your project:

```bash
    dep init
```

This will create a `Gopkg.toml` file specifying your dependencies:

TODO: image here!!!

We have to specify to Heroku what is the location of our project relative to `$GOPATH/src`, so open the `Gopkg.toml` and add at the end of the file the following

```
[metadata.heroku]
  root-package = "local/forgesample"
```


### Git repo setup
On the **forgesample** project root folder create a `.gitignore` file and add the following content:

```
.vscode
Thumbs.db
```

This indicates these files should not be controlled by **git**, as the **.vscode** contains your keys and **thumbs** is created by Windows OS (if you are developing under Windows).

Now initialize **git** for the folder and commit current files. On the terminal (menu **View** >> **Integrated terminal**) type (one line at a time):

```bash
git init
git add .
git commit -m "v1"
```


## Conect to Heroku

Now time to deploy this `v1` of our sample. On the same terminal, sign in to your account:

```bash
heroku login
```

Then create the Heroku app and link with your local folder (one line at a time):

```bash
heroku create forgesample --buildpack heroku/go
heroku git:remote -a forgesample
```

> Heroku app names are unique, so try a different name if you get `Name is already taken` error on **create**.

Now your local **git** knows about the **remote** copy on Heroku. Push changes from your local to remote with:

```bash
git push heroku master
```

## Setup environment variables

It is a good practice to have keys & secrets for local development and for production, so go to your apps on Forge Developer Portal and [create a new app](/account/?id=create-an-app), for instance, **forge sample production**. 

Sign in on [Heroku Dashboard](https://dashboard.heroku.com/) where you app should be listed. Go to **Settings** and create the **Config Vars** as shown on the video below:

![](_media/deployment/heroku/env_vars.gif) 

Ready! You app should be live at your Heroku address, something like: **YourAppName.herokuapp.com**.

## Deploy changes

When you have a new version of your project, login if needed, then just `commit` and `push` live:

```bash
heroku login
git add .
git commit -m "v2"
git push heroku master
```