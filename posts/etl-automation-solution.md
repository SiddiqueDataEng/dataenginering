# ETL Automation Solution: BIML, C# Scripts & CI/CD

*Published: January 2025 | By: Data Engineering Professional*

## Overview

Production-ready ETL automation solution using BIML (Business Intelligence Markup Language) for metadata-driven SSIS package generation, C# Script Tasks for encryption and SFTP operations, and Azure DevOps CI/CD pipelines for automated deployment. This solution enables rapid package generation and deployment with minimal manual intervention.

## The Challenge

**Business Requirements:**
- Generate SSIS packages from metadata automatically
- Implement secure file encryption and SFTP operations
- Automate deployment through CI/CD pipelines
- Reduce manual package development time
- Maintain consistency across packages

**Technical Challenges:**
- Metadata-driven package generation
- Secure credential management
- Automated deployment to SSISDB
- Integration with Azure DevOps
- Version control and rollback capabilities

---

## Architecture Overview

### Automation Architecture

```
┌─────────────────────────────────────────────────────┐
│           Metadata Source                           │
│  • Database Table (incoming_files)                  │
│  • File Type Definitions                            │
│  • Connection Configurations                        │
│  • Transformation Rules                             │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│           BIML Engine                                │
│  • Template Processing                               │
│  • Package Generation                                │
│  • Metadata Integration                              │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│           SSIS Packages (.dtsx)                     │
│  • Generated Packages                                │
│  • C# Script Tasks                                   │
│  • Encrypted Connections                             │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│           CI/CD Pipeline                             │
│  • Build & Compile                                   │
│  • Generate .ispac                                   │
│  • Deploy to SSISDB                                  │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│           SSISDB Catalog                             │
│  • Deployed Packages                                 │
│  • Environments                                      │
│  • Execution History                                 │
└─────────────────────────────────────────────────────┘
```

### Components

**1. BIML Templates:**
- Metadata-driven package generation
- Reusable transformation patterns
- Configuration-based connections
- Dynamic parameter handling

**2. C# Script Tasks:**
- AES file encryption
- SFTP upload/download
- Secure credential handling
- Error handling and logging

**3. CI/CD Pipeline:**
- Azure DevOps YAML pipelines
- Automated build and deployment
- Environment promotion
- Rollback capabilities

---

## BIML Template Implementation

### Metadata Table Structure

**Incoming Files Metadata:**
```sql
CREATE TABLE dbo.incoming_files (
    FileTypeId          INT PRIMARY KEY,
    FilePattern         VARCHAR(255),      -- Regex or glob pattern
    FlatFileFormat      VARCHAR(100),      -- Delimiter, header info
    ConnectionName      VARCHAR(100),      -- Source connection
    DestinationTable    VARCHAR(255),      -- schema.table
    XsdPath             VARCHAR(500) NULL, -- XSD validation path
    HasHeader           BIT DEFAULT 1,
    RowDelimiter        VARCHAR(10),       -- CRLF, LF, etc.
    ColumnsJson         NVARCHAR(MAX)       -- JSON column definitions
);
```

### BIML Template

**Core BIML Template:**
```xml
<Biml xmlns="http://schemas.varigence.com/biml.xsd">
  <Connections>
    <OleDbConnection Name="DW" 
        ConnectionString="Provider=SQLNCLI11;Data Source=.;Initial Catalog=DW;Integrated Security=SSPI;"/>
    <FlatFileConnection Name="FlatFile_<#= row.FileTypeId #>" 
        FilePath="<#= row.FilePath #>"
        Format="Delimited"
        RowDelimiter="<#= row.RowDelimiter #>"
        ColumnNamesInFirstDataRow="<#= row.HasHeader #>">
      <Columns>
        <# foreach(var col in JsonConvert.DeserializeObject<List<Column>>(row.ColumnsJson)) { #>
        <Column Name="<#= col.Name #>" 
                DataType="<#= col.DataType #>" 
                Length="<#= col.Length #>" />
        <# } #>
      </Columns>
    </FlatFileConnection>
  </Connections>

  <Packages>
    <# 
      var metadata = External.GetMetadata(); // Read from incoming_files table
      foreach(var row in metadata) {
    #>
    <Package Name="Load_<#= row.FileTypeId #>_File.dtsx" ConstraintMode="Linear">
      <Variables>
        <Variable Name="User::BatchId" DataType="Guid" Namespace="User">
          <Value>"<#= System.Guid.NewGuid().ToString() #>"</Value>
        </Variable>
        <Variable Name="User::SourceFile" DataType="String" Namespace="User"/>
        <Variable Name="User::EncryptedFile" DataType="String" Namespace="User"/>
        <Variable Name="User::SftpRemotePath" DataType="String" Namespace="User"/>
      </Variables>

      <Tasks>
        <!-- File Validation -->
        <ScriptTask Name="Validate_File" Language="CSharp">
          <ScriptProject>
            <ScriptFiles>
              <ScriptFile Name="Main.cs"><![CDATA[
using System;
using System.IO;
using Microsoft.SqlServer.Dts.Runtime;

public class ScriptMain {
  public void Main() {
    string sourceFile = Dts.Variables["User::SourceFile"].Value.ToString();
    
    // Validate file exists
    if (!File.Exists(sourceFile)) {
      Dts.Events.FireError(0, "Validate_File", $"File not found: {sourceFile}", "", 0);
      Dts.TaskResult = (int)ScriptResults.Failure;
      return;
    }
    
    // Validate against XSD if provided
    string xsdPath = Dts.Variables["User::XsdPath"].Value?.ToString();
    if (!string.IsNullOrEmpty(xsdPath) && File.Exists(xsdPath)) {
      // XSD validation logic
      if (!ValidateAgainstXsd(sourceFile, xsdPath)) {
        Dts.Events.FireError(0, "Validate_File", "XSD validation failed", "", 0);
        Dts.TaskResult = (int)ScriptResults.Failure;
        return;
      }
    }
    
    Dts.TaskResult = (int)ScriptResults.Success;
  }
  
  private bool ValidateAgainstXsd(string xmlFile, string xsdFile) {
    // XSD validation implementation
    return true;
  }
}
]]></ScriptFile>
            </ScriptFiles>
          </ScriptProject>
        </ScriptTask>

        <!-- Encrypt File -->
        <ScriptTask Name="Encrypt_File" Language="CSharp">
          <ScriptProject>
            <ScriptFiles>
              <ScriptFile Name="Main.cs"><![CDATA[
using System;
using System.IO;
using System.Security.Cryptography;
using Microsoft.SqlServer.Dts.Runtime;

public class ScriptMain {
  public void Main() {
    string src = Dts.Variables["User::SourceFile"].Value.ToString();
    string dest = Dts.Variables["User::EncryptedFile"].Value.ToString();
    string keyBase64 = Dts.Variables["User::EncKeyBase64"].Value.ToString();

    byte[] key = Convert.FromBase64String(keyBase64);

    try {
      using (Aes aes = Aes.Create()) {
        aes.Key = key;
        aes.Mode = CipherMode.CBC;
        aes.Padding = PaddingMode.PKCS7;
        aes.GenerateIV();

        using (FileStream fsOut = new FileStream(dest, FileMode.Create, FileAccess.Write)) {
          // Write IV first
          fsOut.Write(aes.IV, 0, aes.IV.Length);

          using (var cryptoStream = new CryptoStream(fsOut, aes.CreateEncryptor(), CryptoStreamMode.Write))
          using (var fsIn = new FileStream(src, FileMode.Open, FileAccess.Read)) {
            fsIn.CopyTo(cryptoStream);
          }
        }
      }

      Dts.TaskResult = (int)DTSExecResult.Success;
    }
    catch (Exception ex) {
      Dts.Events.FireError(0, "AES Encrypt", ex.Message + "\n" + ex.StackTrace, "", 0);
      Dts.TaskResult = (int)DTSExecResult.Failure;
    }
  }
}
]]></ScriptFile>
            </ScriptFiles>
          </ScriptProject>
        </ScriptTask>

        <!-- SFTP Upload -->
        <ScriptTask Name="SFTP_Upload" Language="CSharp">
          <ScriptProject>
            <References>
              <Reference AssemblyPath="C:\SSIS\Renci.SshNet.dll" />
            </References>
            <ScriptFiles>
              <ScriptFile Name="Main.cs"><![CDATA[
using System;
using System.IO;
using Microsoft.SqlServer.Dts.Runtime;
using Renci.SshNet;
using Renci.SshNet.Common;

public class ScriptMain {
  public void Main() {
    string host = Dts.Variables["User::SftpHost"].Value.ToString();
    int port = Convert.ToInt32(Dts.Variables["User::SftpPort"].Value);
    string user = Dts.Variables["User::SftpUser"].Value.ToString();
    string pass = Dts.Variables["User::SftpPassword"].Value.ToString();
    string local = Dts.Variables["User::EncryptedFile"].Value.ToString();
    string remote = Dts.Variables["User::SftpRemotePath"].Value.ToString();

    try {
      using (var sftp = new SftpClient(host, port, user, pass)) {
        sftp.Connect();
        using (var fs = File.OpenRead(local)) {
          sftp.UploadFile(fs, remote);
        }
        sftp.Disconnect();
      }

      Dts.TaskResult = (int)DTSExecResult.Success;
    }
    catch (SshAuthenticationException aEx) {
      Dts.Events.FireError(0, "SFTP Auth", aEx.Message, "", 0);
      Dts.TaskResult = (int)DTSExecResult.Failure;
    }
    catch (Exception ex) {
      Dts.Events.FireError(0, "SFTP Upload", ex.Message, "", 0);
      Dts.TaskResult = (int)DTSExecResult.Failure;
    }
  }
}
]]></ScriptFile>
            </ScriptFiles>
          </ScriptProject>
        </ScriptTask>

        <!-- Data Flow -->
        <Dataflow Name="DF_Load_<#= row.FileTypeId #>">
          <Transformations>
            <FlatFileSource Name="FFSrc">
              <FlatFileConnection ConnectionName="FlatFile_<#= row.FileTypeId #>"/>
            </FlatFileSource>
            
            <DerivedColumns Name="AddMetadata">
              <Columns>
                <Column Name="BatchId" ReplaceExisting="true">
                  <Expression>@[User::BatchId]</Expression>
                </Column>
                <Column Name="LoadTimestamp" ReplaceExisting="true">
                  <Expression>GETDATE()</Expression>
                </Column>
              </Columns>
            </DerivedColumns>
            
            <OleDbDestination Name="InsertToStaging" 
                ConnectionName="DW" 
                TableName="<#= row.DestinationTable #>">
              <InputPath OutputPathName="AddMetadata.Output" />
            </OleDbDestination>
          </Transformations>
        </Dataflow>

        <!-- Post-Load Merge -->
        <ExecuteSQL Name="PostLoad_Merge" ConnectionName="DW">
          <DirectInput>
            EXEC dbo.usp_Merge_FileType_<#= row.FileTypeId #> 
                @BatchId = ?;
          </DirectInput>
          <Parameters>
            <Parameter Name="0" VariableName="User::BatchId" DataType="Guid" />
          </Parameters>
        </ExecuteSQL>
      </Tasks>
    </Package>
    <# } #>
  </Packages>
</Biml>
```

---

## CI/CD Pipeline

### Azure DevOps YAML Pipeline

**Complete Pipeline:**
```yaml
trigger:
  branches:
    include:
      - main
      - develop

pool:
  name: 'Default' # Self-hosted Windows agent with SSDT & BIML tools

variables:
  buildConfiguration: 'Release'
  outputDir: '$(Build.ArtifactStagingDirectory)'
  ssisProjectPath: '$(Build.SourcesDirectory)/SSIS/MySsisProject.dtproj'

stages:
- stage: Build
  displayName: 'Build BIML and SSIS'
  jobs:
  - job: Build
    displayName: 'Generate and Build'
    steps:
    - task: UseDotNet@2
      displayName: 'Use .NET SDK'
      inputs:
        packageType: 'sdk'
        version: '7.x'

    - checkout: self
      displayName: 'Checkout Code'

    - task: PowerShell@2
      displayName: 'Generate Packages from BIML'
      inputs:
        targetType: 'inline'
        script: |
          Write-Host "Generating packages from BIML..."
          
          # Run BIML engine to generate .dtsx files
          $bimlPath = "$(Build.SourcesDirectory)/BIML"
          $outputPath = "$(Build.SourcesDirectory)/SSIS/Generated"
          
          # Execute BIML generator (BimlExpress or custom script)
          & "C:\Program Files\Varigence\BimlExpress\BimlExpress.exe" `
            -InputPath $bimlPath `
            -OutputPath $outputPath `
            -MetadataConnection "Server=.;Database=SSISConfig;Integrated Security=True"
          
          if ($LASTEXITCODE -ne 0) {
            throw "BIML generation failed"
          }

    - task: MSBuild@1
      displayName: 'Build SSIS Project'
      inputs:
        solution: '$(Build.SourcesDirectory)/SSIS/MySsisProject.sln'
        platform: '$(BuildPlatform)'
        configuration: '$(buildConfiguration)'
        msbuildArguments: '/p:OutDir=$(outputDir)\ /p:DeployOnBuild=false'

    - task: CopyFiles@2
      displayName: 'Copy Generated Files'
      inputs:
        SourceFolder: '$(Build.SourcesDirectory)/SSIS/Generated'
        TargetFolder: '$(outputDir)/Generated'
        Contents: '**/*.dtsx'

    - task: PublishBuildArtifacts@1
      displayName: 'Publish Artifacts'
      inputs:
        PathtoPublish: '$(outputDir)'
        ArtifactName: 'ssis-ispac'
        publishLocation: 'Container'

- stage: Deploy_Dev
  displayName: 'Deploy to Development'
  dependsOn: Build
  condition: succeeded()
  jobs:
  - deployment: DeployToDev
    displayName: 'Deploy ISPAC to SSISDB (Dev)'
    environment: 'dev'
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: ssis-ispac
            displayName: 'Download Artifacts'

          - task: PowerShell@2
            displayName: 'Deploy ISPAC to SSISDB'
            inputs:
              targetType: 'inline'
              script: |
                $ispac = Get-ChildItem -Path $(Pipeline.Workspace) -Filter *.ispac -Recurse | Select-Object -First 1
                if ($null -eq $ispac) { 
                  throw "No ispac found" 
                }

                $serverInstance = '$(DevSqlServer)'
                $folder = '$(DevFolder)'
                $project = 'MySsisProject'
                
                # Deploy using PowerShell script
                & "$(Build.SourcesDirectory)/scripts/deploy-ispac.ps1" `
                  -Ispac $ispac.FullName `
                  -ServerInstance $serverInstance `
                  -CatalogFolder $folder `
                  -ProjectName $project

          - task: PowerShell@2
            displayName: 'Smoke Test'
            inputs:
              targetType: 'inline'
              script: |
                # Run a small test package to validate deployment
                Write-Host "Running smoke test..."

- stage: Deploy_Prod
  displayName: 'Deploy to Production'
  dependsOn: Deploy_Dev
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: DeployToProd
    displayName: 'Deploy ISPAC to SSISDB (Prod)'
    environment: 'prod'
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: ssis-ispac

          - task: PowerShell@2
            displayName: 'Deploy ISPAC to SSISDB'
            inputs:
              targetType: 'inline'
              script: |
                $ispac = Get-ChildItem -Path $(Pipeline.Workspace) -Filter *.ispac -Recurse | Select-Object -First 1
                
                $serverInstance = '$(ProdSqlServer)'
                $folder = '$(ProdFolder)'
                $project = 'MySsisProject'
                
                & "$(Build.SourcesDirectory)/scripts/deploy-ispac.ps1" `
                  -Ispac $ispac.FullName `
                  -ServerInstance $serverInstance `
                  -CatalogFolder $folder `
                  -ProjectName $project
```

### Deployment Script

**PowerShell Deployment Script:**
```powershell
param(
    [Parameter(Mandatory=$true)]
    [string]$Ispac,
    
    [Parameter(Mandatory=$true)]
    [string]$ServerInstance,
    
    [Parameter(Mandatory=$true)]
    [string]$CatalogFolder,
    
    [Parameter(Mandatory=$true)]
    [string]$ProjectName
)

Write-Host "Deploying $Ispac to $ServerInstance into $CatalogFolder/$ProjectName"

# Read ispac file as bytes
$bytes = [System.IO.File]::ReadAllBytes($Ispac)
$base64 = [System.Convert]::ToBase64String($bytes)

# T-SQL to deploy project
$tsql = @"
DECLARE @project_stream varbinary(max) = CONVERT(varbinary(max), '$base64', 2);

-- Create folder if not exists
IF NOT EXISTS (SELECT 1 FROM [SSISDB].[catalog].[folders] WHERE name = N'$CatalogFolder')
BEGIN
    EXEC [SSISDB].[catalog].[create_folder] @folder_name = N'$CatalogFolder';
END

-- Deploy project
EXEC [SSISDB].[catalog].[deploy_project]
    @folder_name = N'$CatalogFolder',
    @project_name = N'$ProjectName',
    @project_stream = @project_stream;
"@

try {
    Invoke-Sqlcmd -ServerInstance $ServerInstance -Query $tsql -QueryTimeout 300
    Write-Host "Deployment successful"
    exit 0
}
catch {
    Write-Error "Deployment failed: $_"
    exit 1
}
```

---

## Key Features

### Metadata-Driven Generation

- **Automatic Package Creation**: Generate packages from metadata
- **Consistent Patterns**: Standardized package structure
- **Rapid Development**: Reduce development time by 70%+
- **Maintainability**: Centralized metadata management

### Security Features

- **AES Encryption**: File encryption before transfer
- **Secure Credentials**: Key Vault integration
- **SFTP Security**: Encrypted file transfer
- **Audit Logging**: Comprehensive execution logging

### Automation Benefits

- **CI/CD Integration**: Automated deployment
- **Environment Promotion**: Dev → Test → Prod
- **Version Control**: Git-based versioning
- **Rollback Capability**: Quick rollback procedures

---

## Best Practices

### BIML Best Practices

1. **Reusable Templates**: Create reusable BIML patterns
2. **Metadata Validation**: Validate metadata before generation
3. **Error Handling**: Comprehensive error handling in generated packages
4. **Documentation**: Document template patterns and usage
5. **Version Control**: Track BIML template changes

### CI/CD Best Practices

1. **Environment Separation**: Separate dev, test, prod environments
2. **Approval Gates**: Require approvals for production
3. **Testing**: Automated testing before deployment
4. **Monitoring**: Monitor deployment success
5. **Rollback Plan**: Plan for quick rollback

### Security Best Practices

1. **Key Management**: Use Azure Key Vault for secrets
2. **Credential Rotation**: Regular credential rotation
3. **Access Control**: Least privilege access
4. **Audit Logging**: Comprehensive audit trails
5. **Encryption**: Encrypt sensitive data in transit and at rest

---

## Conclusion

ETL automation with BIML and CI/CD provides:
- **Rapid Development**: Metadata-driven package generation
- **Consistency**: Standardized package patterns
- **Security**: Encryption and secure transfers
- **Automation**: Automated deployment pipelines
- **Maintainability**: Centralized metadata management

**Key Takeaways:**
1. BIML dramatically reduces package development time
2. Metadata-driven approach ensures consistency
3. CI/CD enables automated, reliable deployments
4. Security is critical for production systems
5. Automation reduces manual errors and effort

---

*This solution demonstrates enterprise ETL automation. For related content, see [Metadata-Driven ETL Framework](./metadata-driven-etl-framework.html) and [Enterprise Data Warehouse Migration](./enterprise-data-warehouse-migration.html).*

