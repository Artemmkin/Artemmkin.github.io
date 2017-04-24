---
layout: post
title: >
  How to work with multiple AWS accounts.
tags: [aws, awscli, terr]
---

Ever had to work with multiple AWS accounts? If so, then you probably have a working solution on how to make switching accounts easier, in which case don't hesitate to share it with me in the comments below. But if you still find troublesome managing your multiple AWS credentials, then you should find this post interesting.

If you're used to working with just one AWS account, you most probably used `aws configure` command to quickly set up your _default_ credentials like this.

![200x200](/public/img/terraform/aws-conf.png)  

Recently, I find myself in a situation, when I had to switch between two AWS accounts of two different companies. As it's clearly not a rare case, the first thing I did was to look for a solution on how to manage multiple profiles that Amazon itself would suggest.<!--break-->

And I found that Amazon allows you to create [named profiles](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-multiple-profiles) for each set of your credentials.   

So I created two profiles for each AWS account that I had to work with. I chose to name those profile according to the names of the organizations to which those user accounts belonged.  

<script type="text/javascript" src="https://asciinema.org/a/a86oc30plnxdcfy5weinvrsex.js" id="asciicast-a86oc30plnxdcfy5weinvrsex" async></script>

I forgot to mention that I deleted the default profile which I had previously configured. I did this so that my actions wouldn't affect any AWS resources without me being fully aware of where I'm going to do the changes.

Now, I could use AWS CLI with 2 different user accounts as long as I provided  `--profile <profile_name>` option to each command:
~~~yml
aws ec2 describe-instances --profile ex
~~~
By the way, this command to describe instances gives too big of an output. So I usually use the following alias to get a short description of launched instances:
~~~yml
# ~/.zshrc
alias idesc="aws ec2 describe-instances --query 'Reservations[*].Instances[*].[Placement.AvailabilityZone, State.Name, InstanceId,InstanceType,Tags]' --output text"
~~~
![200x200](/public/img/terraform/idesc.png)  

But having always to provide the profile option in every command can make you quickly tired, right? Good thing, Amazon allows us to use an environment variable to specify the profile we want to use:

![200x200](/public/img/terraform/envvar.png)  

Setting the `AWS_PROFILE` environment variable affects credential loading for all officially supported AWS SDKs and Tools (including the _AWS CLI_ and _Terraform_).

Now new questions arise. The environment variable for a profile is great, but where we define it and how we get information about which profile we're using.

At first, I looked for solutions on the internet, but I didn't find any to my liking. So I came up with a simple bash script which would provide me with ability to quickly switch profiles, turn them off, and give me the visibility into what profile I'm currently using.

Here is how my script looks like:
~~~yml
if [[ $1 = 'on' ]]; then
  if ! aws configure --profile $2 list &> /dev/null ; then
    echo "profile \"$2\" doesn't exist"
  else
    if ! grep "export PS1" ~/.zshrc &> /dev/null ; then
      echo "export AWS_PROFILE=$2" >> ~/.zshrc
      echo "export PS1=\"($2)\$PS1\"" >> ~/.zshrc
    else
      sed -i -e "s/.*export PS1.*/export PS1=\"($2)\$PS1\"/" ~/.zshrc
      sed -i -e "s/.*export AWS_PROFILE.*/export AWS_PROFILE=$2/" ~/.zshrc
    fi
    source ~/.zshrc
  fi

elif [[ $1 = 'off' ]]; then
  sed -i -e '/.*export AWS_PROFILE.*/d' ~/.zshrc
  sed -i -e '/.*export PS1=\(.*\).*/d' ~/.zshrc
  source ~/.zshrc
  unset AWS_PROFILE
else
  echo "Usage:"
  echo "To switch to a specific profile: awspr on profile-name"
  echo "To turn this thing off: awspr off"
fi
~~~

As you can see, I export `AWS_PROFILE` in my `~/.zshrc` file, so that when I choose to switch to a specific profile I could open up new panes in my terminal or even multiple terminal windows and still work the same AWS profile.

I also change `PS1` variable which defines how my command prompt will look like. I add the name of the profile to which I switched at the very beginning of my prompt. This way I can always see what profile I'm using at this moment.

I placed this script under `~/bin` folder (`~/bin/awspr.sh`) and made it executable. Another thing that I did to start using this script was to create a new alias in `~/.zshrc`:
~~~yml
alias awspr=". ~/bin/awspr.sh"
~~~
This launches my script in the _current shell_ when I run `awspr` command.

That's it. Now, to switch to a specific profile I run `awspr on <profile_name>`. And if I don't work with AWS and don't want to see a profile name in the command prompt, I can turn this thing off by running `awspr off`:
<script type="text/javascript" src="https://asciinema.org/a/8j9i3h3hmwb1ghfrgtqclys1v.js" id="asciicast-8j9i3h3hmwb1ghfrgtqclys1v" async></script>

_P.S. the script that I posted here could be easily customized to work with other shells and different linux distributions. My goal was to write something quickly for my personal use._
