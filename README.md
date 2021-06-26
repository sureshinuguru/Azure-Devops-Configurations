# Azure-Devops-Configurations

1)	The build should trigger as soon as anyone in the dev team checks in code to master branch.

      trigger:
        batch: true
        branches:
          include:
          - master

      To clarify this step in the pipeline code, let us say that a push A to master caused the above pipeline to run. While that pipeline is running, additional pushes B and C occur into the repository. These updates do not start new independent runs immediately. But after the first run is completed, all pushes until that point of time are batched together and a new run is started.

      In classic mode:

      Select Enable continuous integration on the Triggers tab to enable this trigger if you want the build to run whenever someone checks in code into master branch

 
2)	 There will be test projects which will create and maintained in the solution along the Web and API.  The trigger should build all the 3 projects - Web, API and test. 
The build should not be successful if any test fails. 

        We can use Path filters on the Build definition and it works flawlessly. I have created 1 build definition per project that needs to live or be hosted somewhere (in my example, I have 3 Build definitions: Web, API, test)
        With the proper path to the project in the Path filter only the proper Builds spin up, and any projects untouched do not trigger a build. Each build has it's own release which then deploys the specified app to it's own destination.



        In azure pipelines.yml for publish test results task , add the last 2 conditions to make build fail when test fails
        task: PublishTestResults@2
          inputs:
            testRunner: VSTest
            testResultsFiles: '**/*.trx'
            failOnStandardError: 'true'  
          failTaskOnFailedTests: 'true'


 
3)	The deployment of code and artifacts should be automated to Dev environment

          - stage: Deploy
             jobs:
              - job: Deploy
                  pool:
                 vmImage: 'vs2017-win2016'
                 steps:

          The target machine is having agent configured to deploy the applications The release definition uses Phases to deploy to target servers.
          Once the build is successful, copy the artifacts and then publish the artifacts in the release will be triggered. Navigate to Releases tab to see the deployment in-progress.

          steps:
          - task: CopyFiles@2
            inputs:
              contents: '_buildOutput/**'
              targetFolder: $(Build.ArtifactStagingDirectory)
          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublishuts: $(Build.ArtifactStagingDirectory)
              artifactName: MyBuildOutp

          - task: PublishPipelineArtifact@1
            inputs:
             #targetPath: '$(Pipeline.Workspace)' # Required

          Deployment task :
          pool:
            vmImage: 'windows-latest'

          variables:
            solution: '**/*.sln'
            buildPlatform: 'Any CPU'
            buildConfiguration: 'Release'

          steps:
           - stage: DeployDevStage
              displayName: 'Deploy App to Dev Slot'
              jobs:
                - job:  DeployApp
                  displayName: 'Deploy App'
                  steps:
                  - task: DownloadPipelineArtifact@2
                    inputs:
                      buildType: 'current'
                      artifactName: 'drop'
                      targetPath: '$(System.DefaultWorkingDirectory)'
                  - task: AzureRmWebAppDeployment@4
                    inputs:
                      ConnectionType: 'AzureRM'
                      azureSubscription: 'Fabrikam Azure Subscription - PartsUnlimited'
                      appType: 'webApp'
                      WebAppName: 'sample'
                      deployToSlotOrASE: true
                      ResourceGroupName: 'sample'
                      SlotName: 'Dev'
                      packageForLinux: '$(System.DefaultWorkingDirectory)/**/*.zip'
4) Upon successful deployment to the Dev environment, deployment should be easily promoted to QA and Prod through automated process.

      Close the same step from Dev release and copy to both QA and Prod and change the host agents/Resource group and subscription for QA and Prod environments for the deployments
      - stage: DeployStagingStage
          displayName: 'Deploy App to Staging Slot'
          dependsOn: DeployDevStage
          jobs:
            - job:  DeployApp
              displayName: 'Deploy App'
              steps:
              - task: DownloadPipelineArtifact@2
                inputs:
                  buildType: 'current'
                  artifactName: 'drop'
                  targetPath: '$(System.DefaultWorkingDirectory)'
              - task: AzureRmWebAppDeployment@4
                inputs:
                  appType: webApp
                  ConnectionType: AzureRM            
                  ConnectedServiceName: 'Fabrikam Azure Subscription - PartsUnlimited'
                  ResourceGroupName: 'rgPartsUnlimited'
                  WebAppName: 'partsunlimited'
                  Package: '$(System.DefaultWorkingDirectory)/**/*.zip'
                  deployToSlotOrASE: true
                  SlotName: 'staging'

5) The deployments to QA and Prod should be enabled with Approvals from approvers only.

      Set Post deployment conditions and Pre deployment conditions like approval mail to approve for the automated deployments in Pre prod and Prod.

      Before:
      - stage: Deploy
      jobs:
      - job: Deploy
      steps:
      - script: echo deploying code

      After:
      - stage: QA
      jobs:
      - job: DeployQa
      steps:
      - script: echo Deploying to QA

      - stage: Production
      jobs:
      - deployment: DeployProduction
      environment: 'Production'
      strategy:
          runOnce:
             deploy:
                 steps:
                    - script: echo Deploying to Production
