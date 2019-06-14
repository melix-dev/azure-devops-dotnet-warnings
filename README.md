# azure-devops-dotnet-warnings
Show build issues (warnings / errors) in Azure Devops for .NET Core projects. 

## The problem
When building .NET core projects in the Azure Devops pipeline with the dotnetcli, the build warnings are not shown in the Build Log / Build Summary views. 

## The solution
### With azure-pipelines.yml
Write the `dotnet build` output to a log file and convert all the issues to a VSO issue. 
This is done in the `azure-pipelines.yml` in this repo. You are free to copy this to your own repo! 

### With classic build pipelines
1.  Add the `.NET Core` restore task to your build pipeline
2.  Add the `.NET Core` build task to your bulid pipeline and update the arguments to: 
    <br/>`--configuration $(BuildConfiguration)  --no-incremental /flp:v=q /flp:logfile=MyLog.log`
    ![image](https://user-images.githubusercontent.com/20542194/59512441-4680b200-8ea8-11e9-821a-89cfe368cb8f.png)
3.  Add the `Powershell` task to your build after the `.NET Core - build` task. 
    Set the `Type` to `Inline`. 
    Copy paste this script to the `Script` field: 
    ```
    $output = Get-Content -Path "$(System.DefaultWorkingDirectory)/MyLog.log";
    $warnings = $output | Select-String -Pattern ".*:\s";
    $hasErrors = $false;
    [regex]$issueTypeRegex = "(warning|error)";
    [regex]$issueLocationRegex = "(\d+,\d+)";
    [regex]$sourcePathRegex = "^[^/(]*";
    [regex]$issueCodeRegex = "(?<=(warning|error) )[A-Za-z0-9]+";
    [regex]$messageRegex = "(?<=((warning|error) [A-Za-z0-9]+: )).*";
    $warnings | Foreach-Object { 
      $issueLocationMatch = $issueLocationRegex.Matches($_)[0];
      $issueLocation = $issueLocationMatch.value.split(",");
      $issueType = $issueTypeRegex.Matches($_)[0];
      $sourcepath = $sourcePathRegex.Matches($_)[0];
      $linenumber = $issueLocation[0];
      $columnnumber = $issueLocation[1];
      $issueCode = $issueCodeRegex.Matches($_)[0];
      $message = $messageRegex.Matches($_)[0];

      Write-Host "##vso[task.logissue type=$issueType;sourcepath=$sourcepath;linenumber=$linenumber;columnnumber=$columnnumber;code=$issueCode;]$message";
      if($issueType.Value -eq "error") { $hasErrors = $true; }
    };
    if($warnings.Count -gt 0 -and $hasErrors -eq $true) { Write-Host "##vso[task.complete result=Failed;]There are build errors"; } 
    elseif($warnings.Count -gt 0 -and $hasErrors -eq $false) { Write-Host "##vso[task.complete result=SucceededWithIssues;]There are build warnings"; } 
      ```
      <br/>
      
      ![image](https://user-images.githubusercontent.com/20542194/59512558-7fb92200-8ea8-11e9-8249-721400699364.png)
      Update the `Control Options: Run this task` to `Even if a previous task has failed, unless the build was cancelled` to make sure that this script will run after a failing build which might be caused by CA errors. 
      ![image](https://user-images.githubusercontent.com/20542194/59512609-9c555a00-8ea8-11e9-9b82-d871d2e0e42a.png)

## Printscreens
Logs tab when only having warnings:<br/>
![image](https://user-images.githubusercontent.com/20542194/59474060-cd944280-8e34-11e9-83b4-2d4db01a330f.png)

Summary tab when only having warnings:
![image](https://user-images.githubusercontent.com/20542194/59474331-d5a0b200-8e35-11e9-9820-db5561fc6289.png)

Logs tab with errors: 
![image](https://user-images.githubusercontent.com/20542194/59497150-62715d00-8e82-11e9-8281-829a0dc21f71.png)

Summary tab with errors: 
![image](https://user-images.githubusercontent.com/20542194/59497339-c8f67b00-8e82-11e9-91d0-331137d43f2d.png)

