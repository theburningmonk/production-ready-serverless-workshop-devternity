# Module 06: CodePipeline and CodeBuild

## Set up a continuous deployment pipeline with CodePipeline and CodeBuild

**Goal:** Have a continous deployment pipeline

<details>
<summary><b>Setup CodePipeline</b></summary><p>

1. Go to the AWS Console

2. Go to the `CodePipeline` console

![](/images/mod06-000.png)

3. Click `Create pipeline`

4. Call your pipeline `workshop-` followed by your name, e.g. `workshop-yancui`

6. Leave the other configurations as is

![](/images/mod06-001.png)

7. Click `Next`

7. Choose `GitHub` as `Source provider`

8. Click `Connect to GitHub`

9. Authorize the app

10. Select your repo, and branch

![](/images/mod06-002.png)

11. Click `Next`

12. Choose `AWS CodeBuild` as `Build provider`

13. Click `Create project` and call the build project `workshop-dev-` followed by your name, e.g. `workshop-dev-yancui`

![](/images/mod06-003.png)

14. Under `Environment image`, choose `Managed image` and choose `Ubuntu`, `Node.js` and `aws/codebuild/nodejs:8.11.0`, and `Always use the latest image for this runtime version`

![](/images/mod06-004.png)

15. Expand `Additional configuration`, under `Environment variables`, add the environment variables `STAGE` and `REGION`. We'll use them to control which stage and region our functions would be deployed to

16. Leave other additional configurations as they are

![](/images/mod06-005.png)

17. Under `Buildspec`, choose `Use a buildspec file`

![](/images/mod06-006.png)

18. Cick `Continue to CodePipeline`

19. Click `Next`

20. Click `Skip`, we'll deploy during the build stage when we run our build script

21. Click `Skip` to confirm

![](/images/mod06-007.png)

22. Review the configuration and click `Create pipeline`

Congratulations! You have created your first pipeline!

Fortunately, this first build is doomed to fail because we haven't created a `buildspec.yml` yet.

![](/images/mod06-009.png)

</p></details>

<details>
<summary><b>Add buildspec.yml</b></summary><p>

1. Add a file `buildspec.yml` under the project's root folder

2. Modify the `buildspec.yml` to the following

```yml
version: 0.2

phases:
  build:
    commands:
      - npm install
      - npm run test
      - npm run sls -- deploy -s $STAGE -r $REGION
      - npm run acceptance
```

3. CodeBuild would require many permissions in order to deploy our serverless project. Go to the IAM console, and look for the role that starts with `code-build-workshop-dev-`.

4. Click `Attach policies`

5. Select `AdministratorAccess` and click `Attach policy`

6. Commit and push your code changes to kick off the pipeline.

If all goes well, you should see the pipeline execute successfully and the functions will be deployed by the pipeline.

![](/images/mod06-011.png)

![](/images/mod06-010.png)

</p></details>

## Exercise

<details>
<summary><b>Separate steps for test, deploy and acceptance</b></summary><p>

Instead of relying on `buildspec.yml` to do everything - integration tests, deploy and acceptance tests - in one giant step, how about we separate them into multiple steps for each environment?

Delete the existing `Build` step, and create a `Dev` that has 3 clear steps:

* IntegrationTest

* Deploy

* AcceptanceTest

![](/images/mod06-016.png)

</p></details>