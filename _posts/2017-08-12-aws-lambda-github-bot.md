---
layout: post
title: >
    How to visualize your workflow with GitHub projects using AWS Lambda.
tags: [aws, awslambda, git, github]
---
### The problem
In our company, we use GitHub for source control of our projects. We have tens of different GitHub repos and almost every one of them has outstanding issues and pull requests (PRs) and as the number of projects grows, it becomes very difficult to manage our work on those projects. Although, we receive notifications about new issues and PRs in our chat, they are not organized. We clearly needed a central place to store and visualize all our issues and PRs, so that we could see the problems we have and prioritize our work

GitHub has [project boards](https://help.github.com/articles/about-project-boards/) that allows you to create Kanban boards for your GitHub issues and PRs. But the problem with this is that the process of adding new issues and PRs to a project board is manual. And we clearly didn't want to go to GitHub and add a new card to the board for every
new issue we receive in our chat.

Thus, we decided to automate this process and create a simple GitHub bot using AWS Lambda.
<!--break-->

### GitHub marketplace

In the GitHub [marketplace](https://github.com/marketplace/category/project-management), I found some project management solutions that could solve our problem. But they literally seemed to cost a fortune.

For example, [ZenHub](https://www.zenhub.com/) creates a board for each repo with all its issues and PRs. It also provides the functionality of placing multiple repositories' issues and PRs [on a single board](https://www.zenhub.com/blog/multi-repo-boards-have-arrived/). However, because boards are created per repository, to see all your issues and PRs across all of your repositories, you would have to go to some repo and merge all your repos to that repo's board, which is kind of not very convenient and what we wanted.

[Codetree](https://codetree.com/) allows your to create a board for opened issues and PRs of multiple repositories and it seemed like it would do for our case, but again the pricing was way too high for our needs.


There is also some other project managements tools like [Waffle](https://github.com/marketplace/waffle) and [Zube](https://github.com/marketplace/zube), but they all cost too much. I believe those could be handy for teams who have projects under intense development with hundreds or even thousands of issues and PRs. However, we don't have any such projects, but we do have plenty of projects some of which have issues and it still requires proper visualization and management.

Besides, it seems like GitHub already has the required functionality in place - the [project boards](https://help.github.com/articles/about-project-boards/). It only required a bit of automation to make the flow of project cards (issues or PRs) across the board visible.

### AWS Lambda

AWS Lambda is a computing service provided by Amazon Web Services (AWS) that lets you create _serverless applications_. With a serverless model, you don't have to worry about provisioning or managing servers (AWS does it for you), and can simply focus on writing your code. Sounds like a dream for any developer, right? :)

In AWS Lambda, you create a function which will be run in response to certain types of events. It is tightly integrated with other AWS services, so you may choose the events to come from many different sources like S3, CloudWatch, SNS, etc.

In our case, we used GitHub integration with Amazon SNS and then made a Lambda function listen to the topic to which the events were pushed. We'll talk more about it in a bit.

A great thing about Lambda is that you pay only for the time your code is run and the amount of memory your function uses. Besides, the Lambda includes 1M free requests per month and 400,000 GB-seconds of compute time per month.

With the current rate of invocations of our 128 MB Lambda function, we use the service for free!

Compared to the paid solutions on GitHub marketplace, we save at least 50 dollars a month.

### Githubotik

It's about time we talk about the Lambda function itself.

I wanted to create something very simple. I wrote [a python module](https://github.com/Artemmkin/aws-lambda-githubbot/tree/master/githubotik/github_functions) to interact with the GitHub API the way I needed. The set of functions that this module provides allowed me to perform different actions with the project cards: creating them, moving across the board, deleting them.

Then I started to think about the Lambda function itself. But before I started to write the code, I had to think about what workflow with GitHub issues and PRs we needed. The idea that I had in mind and which we in the end decided to implement was simple:

_We have one project board called Backlog for opened issues and PRs across all our repositories. The Backlog has 3 columns: TODO, WIP, DONE. For each opened (or reopened) issue or PR a card is created on the board in the TODO column. If someone from our team takes to work on an issue (or PR if it takes too long to merge), he assigns that issue to himself and a card that represents that issue on the Backlog board is moved to the WIP column which stands for "work in progress". When an issue gets closed or a PR gets merged/closed, its card is moved to the DONE column. From the DONE column we would remove the cards manually._

As you can see from the description, our project board was going to be a simple Kanban board for visualizing our workflow with GitHub projects and which would be managed mostly automatically by the Lambda function.

Then I created a Lambda function that would implement the described workflow. I will show a piece of that function. The whole version you can see in my [repo](https://github.com/Artemmkin/aws-lambda-githubbot/blob/master/githubotik/githubotik.py):

~~~python
from config.config_loader import ConfigLoader
from github_functions.githubclient import GithubClient
import json


def lambda_handler(event, context):
    config = ConfigLoader().config()
    github = GithubClient(config['org'], config['token'], config['media_type'])

    gitevent = json.loads(event['Records'][0]['Sns']['Message'])

    if gitevent['action'] == "opened":
        if 'pull_request' in gitevent:
            github.add_pull_request_card(
                config['project'], config['column_for_open'],
                gitevent['repository']['name'],
                gitevent['pull_request']['number'], config['labels'])
        else:
            github.add_issue_card(
                config['project'], config['column_for_open'],
                gitevent['repository']['name'], gitevent['issue']['number'],
                gitevent['issue']['id'], config['labels'])
~~~

You can see here, we first load the ```ConfigLoader``` module. It's another module that I created to load the configuration settings such as the name of the organization, the name of the project board and its columns, token and media type for talking to GitHub API. This ways all the settings that determine the behavior of the Lambda function are stored in a [single json file](https://github.com/Artemmkin/aws-lambda-githubbot/blob/master/githubotik/config/config.json).

Then in the `lambda_handler`, which is the actual Lambda function code that will be executed in response to events, we load the configuration, create an instance of GitHubClient class to talk to GitHub API and load the event from SNS topic. Next we look for different fields in the event, which is basically just a JSON object, and if those fields correspond to the activities that we're waiting for, then we act according to our workflow, i.e. creating or moving the cards across the board.

### Example

I'll show you how you can create a simple Github bot to vizualize your GitHub workflow using code in this [repo](https://github.com/Artemmkin/aws-lambda-githubbot).

The first thing we're going to do is to define the variables in the configuration file that we're loading as part of the Lambda function.
~~~json
{
  "org":  "githubotik-inc",
  "project": "Backlog",
  "column_for_open": "TODO",
  "column_in_progress": "WIP",
  "column_for_closed": "DONE",
  "token": "XXXXXXXXXXXXXXXXXXXXXXXXX",
  "labels": ["help wanted"],
  "media_type": "application/vnd.github.inertia-preview+json"
}
~~~
Most of the settings are pretty much self-explanatory, although I will make a few comments about some of them.

The `media type` should be passed in the `Accept` header of all API calls we make, because the [Projects API](https://developer.github.com/v3/projects/) is currently in the preview period. So this setting may need to be changed in the future.  

The `token` is the GitHub token of a user in your organization on which behalf the automatic actions will be performed. In our organization, it's a special user ```express42-bot```. And in the actual Lambda function we use for ourselves, we don't specify the `token` in the config file, but set it as an environment variable and use KMS encryption with our own key, which technically makes the functions cost us 1 dollar a month :)

You can also specify `labels` that you want to apply to newly made issues and PRs. Although we decided that we didn't need that and turned it off, the functionality is there so you can turn it on if you like.

Next, we will create the `Backlog` project board.

![200x200](/public/img/githubotik/backlog-board.png)


Now we can move on to create an SNS topic in AWS Management Console:

![200x200](/public/img/githubotik/sns-topic.png)

Make sure you copy the topic's ARN, we will need it later.

To allow GitHub to send events to our SNS topic, we need to provide credentials when setting GitHub repo's integration with Amazon SNS. Therefore, as our next step, we create a new user in IAM:

![200x200](/public/img/githubotik/create-user.png)

Make sure your copy the _access_ and _secret_ keys.

As a good security practice, we create a role for our new user giving him only the access he needs. We create a custom inline policy for our new user which will give him rights to publish to the SNS topic we created earlier.

~~~json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "sns:Publish"
      ],
      "Resource": [
        "<SNS topic ARN>"
      ],
      "Effect": "Allow"
    }
  ]
}
~~~

While working with IAM, we also create an AWS Lambda service role for our Lambda function

![200x200](/public/img/githubotik/lambda-role1.png)

and attach `AWSLambdaBasicExecutionRole` managed policy to this role. This will allow our function to write logs to CloudWatch.  

![200x200](/public/img/githubotik/lambda-role2.png)

finally we'll give it a name:

![200x200](/public/img/githubotik/lambda-role3.png)


Now we add an integration with Amazon SNS to one of our organization repositories. To do that, we'll use a [create_hook script](https://github.com/Artemmkin/aws-lambda-githubbot/blob/master/create_hook.py).

Don't try to add the integration manually, because by default GitHub will only send information about _push events_ to the topic and GitHub UI currently doesn't allow you to configure it otherwise, so we have to it via API. And anyway, with this script you can add integration to multiple repos with just one command.

But before we can use it, we need to define some variables.
~~~python
TOKEN = "" # Github token
ORG_NAME = "githubotik-inc" # organization name on Github

## SNS integration configuration
AWS_KEY = ""
AWS_SECRET = ""
SNS_TOPIC = "" # this should be ARN of a topic
SNS_REGION = "" # region where sns topic was created
# see all possible types of events here (https://api.github.com/hooks), look for amazonsns
EVENTS = ["issues", "pull_request"]
~~~

We need to provide a GitHub `TOKEN` to be able talk to GitHub API. As we're going to change the repository's settings, the token has to be provided by someone in your team who has access to the those settings.

`AWS_KEY`,`AWS_SECRET`,`SNS_TOPIC`, `SNS_REGION` variables are values we need to provide to configure a repository integration with Amazon SNS.

`EVENTS` indicates what type of events we want GitHub to send to SNS topic.

After we defined the variables, we now configure our repository with a simple command like this:
~~~bash
$ python create_hook.py nginx-cookbook
~~~

Btw, we can provide multiple arguments to this command to configure multiple repositories.

You can now open a repository's page in your browser to make sure it's configured correctly.

![200x200](/public/img/githubotik/repo-integration.png)

We finally come to the last step which is creating a Lambda function in AWS :)

In order to create a Lambda function in AWS, we need to provide a zip archive of our Lambda function code, as well as all its dependencies.

In this step, we could use some advanced ways to create a Lambda function. For example, the [Serverless](https://serverless.com/) framework looks pretty cool. However, in our simple case, I think it would be unnecessary, as it requires time to install the framework on your machine and learn it. So I decided to go with a good old [Makefile](https://github.com/Artemmkin/aws-lambda-githubbot/blob/master/Makefile) :)

In fact, I took the idea with the Makefile from this [video](https://www.youtube.com/watch?v=68teS9nNvPQ) and it seemed like it would fit perfectly for our project.

When you're done with the code, simply run two commands to package your code and dependencies:

~~~bash
$ make  # create a virtual env
$ make build  # package a function in zip format
~~~

The zip archive will be created in the following path `package/githubotik.zip`.

Let's go now to AWS Management Console once again and create a Lambda function.

Go to AWS Lambda service page and look for `sns` blueprint.

![200x200](/public/img/githubotik/aws-lambda1.png)

Add sns topic we created earlier as the trigger and enable it:

![200x200](/public/img/githubotik/aws-lambda2.png)

Set the name of the function and choose the runtime environment:

![200x200](/public/img/githubotik/aws-lambda3.png)

Choose to upload a zip package and upload our function:

![200x200](/public/img/githubotik/aws-lambda4.png)

Fill in the `Handler` field according to the instructions and select the role for our function:

![200x200](/public/img/githubotik/aws-lambda5.png)

Click advanced settings to configure the memory usage and timeout. For our simple function, 128 MB of memory will be more than enough and it hardly ever takes to run longer than 5 seconds.

![200x200](/public/img/githubotik/aws-lambda6.png)

That's it. Now let's try it out! In the video below, I'll demonstrate how the work with issues and PRs is now visualized on the project board:
<br><br>

<iframe width="560" height="315" src="https://www.youtube.com/embed/UU3TM-hG9tg" frameborder="0" allowfullscreen></iframe>

### Conclusion

Another problem that we faced when we started to use this Github bot was that we had to put all our existing issues and PRs on the board. If you decide to use it, you may find useful [this](https://github.com/Artemmkin/aws-lambda-githubbot/blob/master/add_old_issues_to_project.py) script which does it automatically for you.

We've been using this bot for over a month now and so far it really made our work with Github projects visible and more manageable.
