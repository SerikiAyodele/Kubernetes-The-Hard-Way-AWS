PREREQUISITES
1. install the AWS CLI {https://aws.amazon.com/cli/}
![aws cli](img/1.png)

2. Set a default
```
AWS_REGION=us-west-1

aws configure set default.region $AWS_REGION
```

3. I installed tmux for running commands in parallel on multiple compute instances at the same time. this will help to speed up the provisioning process
`brew install tmux`
![tmux](img/2.png)