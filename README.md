parameters:
- name: Configuration
  displayName: Configuration
  type: string
  default: Full
  values:
  - Full
  - GroupWise
- name: testFiltercriteria
  displayName: Test Filter criteria
  type: string
  default: Group=ApplyFilter
  values:
  - Group=ESAUTO1
  - Group=ApplyFilter
  - Group=Onboarding
  - Group=TicketPreview
  - Group=EscalationWizard
  - Group=OpenEsc
  - Group=Security
  - Group=TicketSearch
  - Group=WatchList
  - Group=TicketSummary
  - Group=Fishbone_Tool
  - Group=TeamMeetingScheduler
trigger:
  branches:
    include:
    - none
variables:
- name: BuildConfiguration
  value: 'Release'
- name: BuildPlatform
  value: 'Any CPU'
- name: JAVA_HOME
  value: C:\Program Files\Java\jdk-17\java-17
- name: Configuration
  value: Full,GroupWise
stages:
- stage: Build
  jobs:
  - job: Build
    pool:
      name: LamVMSSAgentPool
    steps:
    - task: 6d15af64-176c-496d-b583-fd2ae21d4df4@1
      inputs:
        repository: self
    - task: SonarQubePrepare@5
      displayName: Prepare Sonar Analysis
      inputs:
        SonarQube: 'Sonar'
        scannerMode: 'CLI'
        configMode: 'manual'
        cliProjectKey: 'CSBG_ESRedesignAutomation'
        cliProjectName: 'CSBG_ESRedesignAutomation'
        cliSources: '.'
    - task: NodeTool@0
      displayName: Install Node.Js
      inputs:
        versionSource: 'spec'
        versionSpec: '16.x'
    - task: PowerShell@2
      displayName: 'Install JDK 17'
      inputs:
        targetType: inline
        script: "# Define the URL of the zip file and the destination paths\n$zipUrl = \"https://qa-artifactory.lamresearch.com:443/artifactory/devops-generic-testing-lwestus/Java-17/Windows/Java-17.zip\"\n$destinationPath = \"C:\\Program Files\\Java\\jdk-17\"\n$tempZipPath = \"C:\\temp\\Java-17.zip\"\n  \n# Create the destination directory if it does not exist\nif (-Not (Test-Path -Path $destinationPath)) {\n    New-Item -ItemType Directory -Force -Path $destinationPath\n}\n  \n# Create the temporary directory if it does not exist\n$tempDir = [System.IO.Path]::GetDirectoryName($tempZipPath)\nif (-Not (Test-Path -Path $tempDir)) {\n    New-Item -ItemType Directory -Force -Path $tempDir\n}\n  \n# Download the zip file\nInvoke-WebRequest -Uri $zipUrl -OutFile $tempZipPath\n  \n# Extract the zip file to the destination path\nExpand-Archive -Path $tempZipPath -DestinationPath $destinationPath -Force\n\n# Clean up the temporary zip file\nRemove-Item -Path $tempZipPath -Force\n  \nWrite-Host \"Java 17 has been downloaded and extracted to $destinationPath\"\n"
    - task: CopyFiles@2
      displayName: File Copy
      inputs:
        SourceFolder: ''
        contents: '**'
        targetFolder: $(Build.ArtifactStagingDirectory)
    - task: ArchiveFiles@2
      displayName: Archive Artifacts
      inputs:
        rootFolderOrFile: '$(Build.ArtifactStagingDirectory)'
        includeRootFolder: false
        archiveType: 'zip'
        archiveFile: '$(Build.ArtifactStagingDirectory)/artifacts.zip'
    - task: PublishBuildArtifacts@1
      displayName: Publish Artifacts
      inputs:
        PathtoPublish: '$(Build.ArtifactStagingDirectory)/artifacts.zip'
        ArtifactName: 'drop'
        publishLocation: 'container'
- stage: DeployDev
  displayName: 'Deploy to Dev'
  variables:
  - group: ESRedesignAutomation_Server_Details_Dev
  - group: dcazkmweb01_Server_Credentials
  jobs:
  - job: Deploy
    pool:
      name: LamVMSSAgentPool
    steps:
    - task: DownloadBuildArtifacts@1
      displayName: Download Artifact
      inputs:
        buildType: 'current'
        downloadType: 'single'
        artifactName: 'drop'
        downloadPath: '$(System.ArtifactsDirectory)'
    - task: PowerShell@2
      displayName: Unzip Artifacts
      inputs:
        targetType: inline
        script: |
          Expand-Archive -Path "$(System.ArtifactsDirectory)\drop\artifacts.zip" -DestinationPath "$(System.ArtifactsDirectory)\drop"
    - task: WindowsMachineFileCopy@2
      displayName: Deploy
      inputs:
        SourcePath: '$(System.ArtifactsDirectory)\drop\ESAutomation'
        MachineNames: '$(ServerName)'
        AdminUserName: '$(Username)'
        AdminPassword: '$(Password)'
        TargetPath: '$(ApplicationPath)'
        Overwrite: true
- stage: GroupWiseTest
  displayName: 'GroupWise Test Scripts Execution'
  jobs:
  - job: GroupWise
    timeoutInMinutes: 1440
    pool:
      name: LamAgentPool_Interactive
    steps:
    - task: 6d15af64-176c-496d-b583-fd2ae21d4df4@1
      inputs:
        repository: self
    - task: VisualStudioTestPlatformInstaller@1
      displayName: Visual Studio Test Platform Installer
      inputs:
        packageFeedSelector: 'nugetOrg'
        versionSelector: 'latestPreRelease'
    - task: InstallTestCompleteAdapter@1
      displayName: TestExecute test adapter installer
      inputs:
        installExecutor: false
        accessKey: '11eeea6d-e109-459f-b08a-099569cf82c0'
        logsLevel: '0'
    - task: UsePythonVersion@0
      inputs:
        versionSpec: '3.x'
        addToPath: true
        architecture: 'x64'
    - task: CmdLine@2
      displayName: Install Required Python Libraries
      inputs:
        script: |
          python3.11 -m venv venv
          source venv/bin/activate
          pip install pytz --target="C:\Program Files (x86)\SmartBear\TestExecute 15\x64\Bin\Extensions\Python\Python311\Lib"
          python -m pip install requests --target="C:\Program Files (x86)\SmartBear\TestExecute 15\x64\Bin\Extensions\Python\Python311\Lib"
          pip install chardet --target="C:\Program Files (x86)\SmartBear\TestExecute 15\x64\Bin\Extensions\Python\Python311\Lib"
          pip install charset-normalizer==2.1.0 --target="C:\Program Files (x86)\SmartBear\TestExecute 15\x64\Bin\Extensions\Python\Python311\Lib"
    - task: VSTest@3
      displayName: VsTest - testAssemblies
      condition: ''
      inputs:
        testSelector: 'testAssemblies'
        vsTestVersion: 'toolsInstaller'
        testAssemblyVer2: '**\ESAutomation.pjs'
        searchFolder: '$(System.DefaultWorkingDirectory)'
        testFiltercriteria: Group=ApplyFilter
- stage: CompleteTest
  displayName: 'Complete Test Scripts Execution'
  jobs:
  - job: CompleteTest
    timeoutInMinutes: 1440
    pool:
      name: LamAgentPool_Interactive
    steps:
    - task: 6d15af64-176c-496d-b583-fd2ae21d4df4@1
      inputs:
        repository: self
    - task: VisualStudioTestPlatformInstaller@1
      displayName: Visual Studio Test Platform Installer
      inputs:
        packageFeedSelector: 'nugetOrg'
        versionSelector: 'latestPreRelease'
    - task: InstallTestCompleteAdapter@1
      displayName: TestExecute test adapter installer
      inputs:
        installExecutor: false
        accessKey: '11eeea6d-e109-459f-b08a-099569cf82c0'
        logsLevel: '0'
    - task: UsePythonVersion@0
      inputs:
        versionSpec: '3.x'
        addToPath: true
        architecture: 'x64'
    - task: CmdLine@2
      displayName: Install Required Python Libraries
      inputs:
        script: |
          python3.11 -m venv venv
          source venv/bin/activate
          pip install pytz --target="C:\Program Files (x86)\SmartBear\TestExecute 15\x64\Bin\Extensions\Python\Python311\Lib"
          python -m pip install requests --target="C:\Program Files (x86)\SmartBear\TestExecute 15\x64\Bin\Extensions\Python\Python311\Lib"
          pip install chardet --target="C:\Program Files (x86)\SmartBear\TestExecute 15\x64\Bin\Extensions\Python\Python311\Lib"
          pip install charset-normalizer==2.1.0 --target="C:\Program Files (x86)\SmartBear\TestExecute 15\x64\Bin\Extensions\Python\Python311\Lib"
    - task: VSTest@3
      displayName: VsTest - testAssemblies
      condition: ''
      inputs:
        testSelector: 'testAssemblies'
        vsTestVersion: 'toolsInstaller'
        testAssemblyVer2: '**\ESAutomation.pjs'
        searchFolder: '$(System.DefaultWorkingDirectory)'


