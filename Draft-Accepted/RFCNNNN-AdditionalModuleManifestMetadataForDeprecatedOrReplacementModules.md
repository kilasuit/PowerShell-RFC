---
RFC: RFC<four digit unique incrementing number assigned by Committee, this shall be left blank by the author>
Author: Ryan Yates (@kilasuit)
Status: Draft
# SupercededBy: 
Version: 0.1
Area: Module Manifests & how they work with Package Registries like PowerShell Gallery
Comments Due: April 4th 2026
Plan to implement: Yes but with help
Helpers:
- PSGalleryTeam
- PowerShell/PSScriptAnalyzer
- PowerShell/PSResourceGet
- CmdletsWG / EngineWG
---

# Title

Extend Module Manifest Capabilities to enable queryable data around deprecations and replacements.

## Description

Add some additional recommended properties in Module Manifests to aid Module Authors, Systems Administrators & Developers that hopefully keeps these in a part of the Module Manifest that should require no changes within the engine, unless we were to choose to add additional engine checks at a later date.

By adding more deterministic properties that are documented as being added to either the PSData or PrivateData section in the Module Manifest we can gather and show this information to users whether in PSResourceGet or in the PowerShell Gallery or custom tooling.

The important part of adding these properties is that the links that we can create with these new properties can be either bi-directional or one-direction only.


## Motivation

    As a module author I require a simple mechanism to allow me to determine when & how I should be replacing any of my module's dependencies, particularly if there are security concerns or a dependency is seemingly no longer being maintained.

    This should start with PSModules and be expandable to PSScripts and then other dependencies like Assemblies and Executables in future revisions.

    Today this is both hard to determine and lacks the ability to build a sensible graph around the functionality that is similar or needs extensive work to accommodate dropping in replacement dependencies.

    By Implementing this and adding additional tooling safeguards and checks this would help users of my modules as well as users of other modules in this painful issue that impacts all manners of package registries and should help make Module Management easier in future.

    Also as a module author I may decide to deprecate a module that I have authored that I no longer wish to continue working on & require a metadata based mechanism to be able to inform downstream users of my code of this decision.

## User Experience

Whether in Interactive or Scripted non-interactive use we should be able to return information that can be easily actioned on.

```powershell
Install-PSResource DeprecatedModule
```

```output
Warning: Module DeprecatedModule has been flagged has having potential replacements either by the Module Author or by other Module Authors that previously depended on it.
RecommendedAction: Review this combined list of ReplacementModules with their reasons for replacing them and update where possible to do so to one of them.
```

```powershell
Install-PSResource ModuleDependingOnDeprecatedModule
```

```output
Warning: Module ModuleDependingOnDeprecatedModule previously depended on a now marked as deprecated module, either by the Module Author or by other Module Authors that previously depended on it.
RecommendedAction: Review this combined list of ReplacementModules with their reasons for replacing them and update where possible to do so to one of them.
```

Searches on the PowerShell Gallery could then show either a Deprecated Banner or a ReplacedBy banner highlighting this new functionality.

In time a full redesign of the Gallery's frontend and how data gets exposed may actually make sense but that conversation is out of scope for this RFC.

## Specification

The below is a detailed example of how this could be implemented as a number of new properties to be included in a resulting manifest in different ways. 

Exactly where this needs adding & how to make it simple via automated means is brought up as part of this RFC up for debate.

I am open to all discussions on how much of the below that we choose not to implement as much of it may not be of real use and be too much bloat for it to actually catch on and be used appropriately. However, for completeness of a first pass draft I felt it worthwhile showing differing implementation options. 

The below was pulled together essentially without any testing over the space of a few days between other things & is intentionally rough around the edges but with enough comments contained within to allow getting the widest range of suggestions & feedback going forward.

```powershell
DeprecatedDetails  = [PSCustomObject]@{
            DeprecatedReason = ''
            ## Ideally this would be filled out with the details of the new module to use however none of this is mandatory, as to allow a simple one off deprecated update. 
            # Optionally a Module Author at point of deprecating a module can provide the following information helping point users to other modules that contain similar functionality. 
            ReplacementModuleDetails = @{
                Name = ''
                # I initially debated whether this should include Author or CompanyName or potentially both & decided that both should be included.
                Author = ''
                CompanyName = ''
                GUID = ''
                # Version needs to be an explicit Required Version & not a Module Range or Minimum Version to avoid potential issues caused by any accidental or planned breaking changes to that module in future versions. It is then up to the module author to determine how they wish to handle the dependency update process for this replacement module.
                # Module Authors can then determine if they want to depend on a more recent version
                # Can include preview versions but recommended where possible not to use a preview version.
                Version = ''
                # Ideally this is a link to the main project/repository page for the module where the source code lives & if this is not provided a warning should be shown to module authors during their dev process. 
                ProjectUri = ''
                # This block should allow for both PowerShellGet's Register-PSRepository & PSResourceGet's Register-PSResourceRepository to register the repository where this replacement module can be found and installed from.
                Repository = @{
                    # If Name is PSGallery, MAR, NuGet, or any other well known repository then we would not need Uri,Type,Priority,AuthType as these are already likely configured or can be easily configured using defaults for these well known repositories. In time we may be able to have a set of known repository shortnames as well as importable registry lists that can be used here to further simplify the management of these in larger environments.
                    # For more on this see https://github.com/PowerShell/PSResourceGet/issues/1883
                    Name = ''
                    Uri = ''
                    Type = ''
                    Priority = ''
                    # Ideally AuthType would be 'Anonymous' for public repositories but could be other types for private/internal repositories
                    AuthType = 'Anonymous' # Other AuthTypes include 'Basic', 'OAuth', 'PersonalAccessToken', 'Windows' & 'ApiKey' - more details can be found in the Register-PSRepository & Register-PSResourceRepository documentation
                    # Additional fields would be needed here depending on which of the AuthTypes is listed.
                }
                # If Repository cannot be provided, at the very least a direct link to where the module package that is in an archived format and can be downloaded from should be provided here.
                PackageURI = ''
                # If this is not possible provide some form of contact method for the module author/maintainer to gain a module package via other means.
                PackageContactDetails = ''
            }
            # Here I go into detail about how we may instead condense this.
            ## To do this we build a Fully Unique ID (FUID) instead of the above. this is to enable quicker searching and matching across various repositories should the module name be similar to others and to reduce maintenance overhead
            ## FUID is a concept that is detailed in https://blog.kilasuit.org/2025/09/24/announcing-fuid/ and is more useful than a GUID/UUID and is more like defining a Resource in Bicep. 
            # The above field and its sub properties should be able to be condensed into a single deterministic string (a FUID) that can not only be parsed for the relevant details but can also be hashed if needed too. We use the '|' character as a top level delimiter & ';' character as a delimiter for any lower levels which should ensure proper grouping of related information to provide a non-breakable format that is both human & machine readable as well as being relatively easy for rebuilding into a more complex object as required, however more data is needed from PowerShell Gallery to determine if this format will be reliable or may need some tweeking.
            # Example of this format as a FUID String for Microsoft.PowerShell.ThreadJob replacing PoshRSJob or ThreadJob would be
            # 'PSModule:Microsoft.PowerShell.ThreadJob|Microsoft Corporation|12345678-90ab-cdef-1234-567890abcdef|2.0.0|GH:PowerShell/ThreadJob|PSGallery'
            ReplacementModuleFUID = ''
            # Other options here include providing a hash of the above ReplacementModuleFUID string to allow for quicker matching without needing to parse the entire string each time but removes much of the human readability. 
            ReplacementModuleFUIDHash = ''
            # Or we could provide both to allow for both human & machine readability.
            # We can also potentially choose instead provide an encrypted version of the ReplacementModuleFUID to ensure authenticity of the data provided here that can be verified using either a verifiable public key, like SSH, PGP or a Certificate based approach
            ReplacementModuleFUIDEncrypted = ''
            ReplacementModuleFUIDEncryptedType = ''
            ReplacementModuleFUIDEncryptedPublicKey = ''
            # From here you can get the gist of where I am going with this - providing multiple ways to verify & validate the replacement module details to ensure users can trust the information being provided here
            # You could also provide a list of multiple replacements in case there are several modules that can be used as replacements
            MultipleReplacements = @{
                # Here you would provide a list of replacement modules with the same structure as above but also adding a suggested order of preference for users to follow providing a preference reason for each replacement
                # Example below
                # This format should in most cases be easily pulled from a module manifest, without needing external network calls to complete.
                # It is possible to generate it from Find-Module or in future too when Find-PSResource returns some of the missing AdditionalMetadata properties that it uses, notably GUID here.
                # In time I'd actually like this to be autogenerated and added in a Module Manifest when people run either New-ModuleManifest or Update-PSModuleManifest
                # I intend to release a Get-ModuleFUID function to create these. Unless it's more sensible to add this as part of core PowerShell functionality and also release it for downlevel access via the Gallery or as a dependency to PSResourceGet
                Replacements = @(
                    [PSCustomObject]@{
                        ReplacementModuleFUID = 'PSModule:Microsoft.PowerShell.Archive|Microsoft Corporation|Microsoft Corporation|12345678-90ab-cdef-1234-567890abcdef|2.0.0|GH:PowerShell/ThreadJob|PSGallery'
                        PreferenceReason = 'TeamDeveloped & Maintained'
                        PreferenceOrder = 1
                    },
                    [PSCustomObject]@{
                        ReplacementModuleFUID = 'PSCompression|santisq|Unknown|c63aa90e-ae64-4ae1-b1c8-456e0d13967e|3.0.1|GH:santisq/PSCompression|PSGallery'
                        PreferenceReason = 'Community Developed & Well Maintained, Extensive Feature Set with additional useful features not currently available in Microsoft.PowerShell.Archive'
                        PreferenceOrder = 2
                    }
                )
                # As you can see this format is much more condensed so preferable to the expanded format shown above.
            }
        }
        ReplacesDetails = [PSCustomObject]@{
            # Similarly in here we could have as much or as little info in this section as is shown above
            ReplacementReason = ''
            DeprecatedModuleDetails = @{
                Name = ''
                Author = ''
                CompanyName = ''
                GUID = ''
                # Version needs to be an explicit Required Version & not a Module Range or Minimum Version to avoid potential issues caused by any accidental or planned breaking changes to that module in future versions. It is then upto the module author to determine how they wish to handle the dependency update process for this replacement module.
                Version = ''
                # Ideally this is a link to the main project/repository page for the module where the source code lives & if this is not provided a warning should be shown to module authors during their dev process.
                ProjectUri = ''
                # This block should allow for both PowerShellGet's Register-PSRepository & PSResourceGet's Register-PSResourceRepository to register the repository where this replacement module can be found and installed from.
                Repository = @{
                    Name = ''
                    Uri = ''
                    Type = ''
                    Priority = ''
                    # Ideally AuthType would be 'Anonymous' for public repositories but could be other types for private/internal repositories
                    AuthType = 'Anonymous' # Other AuthTypes include 'Basic', 'OAuth', 'PersonalAccessToken', 'Windows' & 'ApiKey' - more details can be found in the Register-PSRepository & Register-PSResourceRepository documentation
                }
                # If Repository cannot be provided, at the very least a direct link to where the module package that is in an archived format and can be downloaded from should be provided here.
                PackageURI = ''
                # If this is not possible provide some form of contact method for the module author/maintainer
                PackageContactDetails = ''
            }
            # In time the above should be able to be condensed into a single deterministic string that can not only be parsed for the relevant details but can also be hashed too. We use the '|' character as a top level delimiter & ';' character as a delimiter for any lower levels which should ensure proper grouping of related information to provide a non-breakable format that is both human & machine readable.
            # Example using PoshRSJob which technically has been feature replaced by the Microsoft.PowerShell.ThreadJob Module
            # Generated using the below code.
            # $PoshRSJob = find-module PoshRSJob
            # $PoshRSJobFUID = "PSModule:$($PoshRSJob.Name)|$($PoshRSJob.Author)|$(if ($PoshRSJob.AdditionalMetadata.CompanyName -eq $null) {'NoCompanyNameProvided'} else {$PoshRSJob.AdditionalMetadata.CompanyName} )|$($PoshRSJob.AdditionalMetadata.GUID)|$($PoshRSJob.Version)|$(if ($PoshRSJob.ProjectURI.OriginalString -match 'github') {"GH:$($PoshRSJob.ProjectURI.OriginalString.replace('https://github.com/',''))"} elseif ($PoshRSJob.ProjectURI.OriginalString -eq $null) {'NoProjectUri'} else {$PoshRSJob.ProjectURI.OriginalString} )|PSGallery"
            # $PoshRSJobFUID | Set-Clipboard
            # Note due to differences in the Manifest vs what Find-Module returns for some of the toplevel properties we require digging into the Additional Metadata
            DeprecatedModuleFUID = 'PSModule:PoshRSJob|Boe Prox|NA|9b17fb0f-e939-4a5c-b194-3f2247452972|1.7.4.4|GH:proxb/PoshRSJob|PSGallery'
            }
            MultipleDeprecated = [PSCustomObject]@{
                Deprecations = @(
                    [PSCustomObject]@{
                        DeprecatedModuleFUID = 'PSModule:Microsoft.PowerShell.Archive|Microsoft Corporation|Microsoft Corporation|eb74e8da-9ae2-482a-a648-e96550fb8733|1.2.5|NoProjectUri|PSGallery'
                        DeprecatedReason = @('Archived by team and not actively maintained', 'Missing ProjectUri')
                    },
                    [PSCustomObject]@{ 
                        # Picking on this as it was not updated since 2018.
                        DeprecatedModuleFUID = 'PSModule:ziphelper|Kirill Kravtsov|NoCompanyNameProvided|97dac4a3-82a1-42e0-aec4-1aee66797120|0.2.2|GH:nvarscar/psziphelper|PSGallery'
                        DeprecatedReason = @('Author has not updated this for x duration and as such should be seen as abandoned', 'Author has also not committed much other public code in x duration, so may not be wanting to continuing supporting this code', 'Missing CompanyName') 
                    }
                )
                # As you can see this format is much more condensed so preferable to the expanded format option shown above.
            }
```

## Alternate Proposals and Considerations

I don't feel that there really is a good mechanism today for many module authors either deprecating one of their modules or adding potential replacements for things they had used previously that isn't pot luck in getting the right information at the right time.

I think many will not have the time or motivation to author a Proxy Module, even if they know of their existence.  

I know I personally wouldn't do so to act as a bridge between the old and new as this does provide delays & headaches that for many are best avoided if possible. This also doesn't stop that mechanism from being used if deemed the right course of action but would aid  discovery at Install time instead of it being something that needs dug into release notes, or project repositories.

With tooling like Copilot as well as Security Scanning tools as well as dependency tracking tools like Dependabot adding this functionality this way allows those tools to make use of this updated set of properties to hasten the replacement processes.

We could recommend adding additional files in modules that capture this data, but it's better served as metadata in this manner.

My Only other real alternative proposal whilst semi linked to this was to also add runtime checks in the Engine, which at this time still would need these properties to be available to be able to check for them during tasks like at Module Import or building the CommandCache. Neither of which feel like they make sense, at least not yet.

Lastly the final alternative proposal was to just not do anything. Which is not suitable for attempting to resolve or lessen the impact felt from this issue.

### Consideration 1

I don't think that we can just rely on functionality in Nuget for this functionality as whilst this may be retrievable via nuget this is data that could / should be contained in a module manifest anyway for all other ways of performing data analysis of the content within them. 

We should also equate for how this would be of use in environments that are locked down, as well as in organisations that are making use of already available scanning tools including those that may be using content scanning mechanisms that do not make use of any PowerShell functionality to do this, like Regex Pattern Matching for example.

### Consideration 2

The Gallery has lots of dead/abandoned modules and modules missing key metadata, whilst it would be nice to be able to remove lots of these modules after issuing some form of notice requesting an update prior to removal this is not really feasible.

As such this is out of scope for this RFC.

However, by Implementing this functionality over time we could see an improvement via a reduction of Modules in the Gallery that tools like PSResourceGet & the PowerShell Gallery Search return for queries, to only those that have not been marked as Deprecated or Replaced.

That aspect is a future implementation consideration and is at this time intentionally out of scope for this RFC.

### Consideration 3

The Gallery is now over 10 years old and as such is likely running on a codebase that whilst has been largely kept running with minor functional changes, is not ideal to maintain. It's architecture potentially hasn't changed all that much either since it was originally deployed. However, intentionally much of this detail is internal to the team & rightly so.

Whilst in many ways it would be nice to build a brand new PowerShell Gallery which takes Consideration 2 into account as well as this one, with discussions on what new or improved features the community would like to see, this is such a task this is out of scope for this RFC for the community to discuss even if there items, like deprecated & replaced banners that could be part of that discussion. However the team can decide to have internal discussions around the viability of those 2 particlar minor enhancements. 

### Consideration 4

Getting a good design on this is crucial and having only a 1 month period for comments is not long enough, whether the work for this is managed in the community or partly managed by the team. Also the time of raising this RFC is at the end of the year and may be missed if not left for the extended period chosen.

### Consideration 5 

We may want a mechanism built into the Gallery, as well as via PSResourceGet to allow us to attach a Deprecation/Replacement Notice to Packages hosted on the Gallery, but is external to the process being suggested here. 

This is currently not deemed realistically feasible to implement at this time and whilst a nice to have, that is currently deemed also out of scope for this RFC & may be suggested in another in the future.

### Consideration 6 

Following on from Consideration 4 the amount of time I have available towards stuff like this is significantly much less than this time last year. As such I may only really be able to help steer the High-level design during this RFC process, and likely will need to leave the low level and implementation details to others to ensure this gets implemented in a timely manner.

This does not need to mean that this has to be left to the team to implement, but may be a worthwhile project to run as a side community hackathon session at any of the upcoming PowerShell conferences in 2026.

Whilst I would like to be able to say I absolutely want to be able to spend the time needed to  implement this, I have to be realistic that I will not have the time to do so.

This is why I prior to raising this I submitted an amendment to the New-RFC-Template with the Helpers section at the top as well as the Required Help section below.

### Consideration 7

Whilst this has most potential to impact the `ModuleManifest` Cmdlets and Functions available across PowerShellGet, PSResourceGet & as part of Microsoft.PowerShell.Core there are many other ways that manifests are created today as well as how this information may want to be returned in `System.Management.Automation.PSModuleInfo` as new alias properties in a future release.

The cmdlets and functions are

```powershell

Name                    Source                             Version   CommandType
----                    ------                             -------   -----------
Update-ModuleManifest   PowerShellGet                      2.2.5        Function
New-ModuleManifest      Microsoft.PowerShell.Core          7.5.0.500      Cmdlet
Test-ModuleManifest     Microsoft.PowerShell.Core          7.5.0.500      Cmdlet
Update-PSModuleManifest Microsoft.PowerShell.PSResourceGet 1.1.1          Cmdlet

```

By keeping this as part of either PrivateData or PSData we can remove some of the potential implementation headaches as these fields mostly seem to allow for additional properties to be added to Module Manifests by Module Authors without it having much if any noticeable impact to how the PowerShell Engine & the CommandCache is built and maintained.

However I may have misunderstood this as part of my investigations.

### Consideration 8

This likely should be wrapped in an experimental feature, especially if deemed that `System.Management.Automation.PSModuleInfo` or the cmdlets in `Microsoft.PowerShell.Core` need updating, however that may be best as a second pass implementation attempt, however they would not be usable downlevel so it may only be possible for `Update-PSModuleManifest` to include additional parameters for whatever fields are decided to be included, particularly if determined to be best placed in PSData, not Private Data.

### Consideration 9

This will need to be well publicised once added, as to get this new metadata included in new modules and versions so that it can be used properly in future.

We may see benefit with attempting to do a wide ranging scan of the modules in the Gallery and attempt to by using this data send requests for the addition of this new metadata to existing packages. However this likely may not get as much traction as desired for the effort but is worth considering whether this in or out of scope or is a post implementation action that the team or the community or a collaboration across both the team and community want to spend the time on attempting over many months or even years.

### Consideration 10

Additional tooling like adding a PSScriptAnalyzer Rule or useful Pester Tests may be beneficial to think about but may not provide much value outside of these being added as part of existing testing suites.

A potential PSScriptAnalyzer Rule may be to check Module dependencies and warn if any of them get flagged during build & testing of a projects module manifest. This could be wrapped in a Pester test to better surface this newly available information in future in CI/CD tooling but also could be surfaced during development via the existing VSCode & PSScriptAnalyzer features.

I think this is worth calling out for consideration, but may be best done as post implementation tasks.

### Consideration 11

Many existing modules, particularly those a part of Windows or from other teams both in and outside of Microsoft aren't always available via the Gallery or a Public Repository of any kind so updates to these are challenging. 

Many modules even available via the Gallery also don't tend to list all of the Required Modules & their versions as dependencies either, opting in many cases for these to only be modules that are available via the Gallery, so these dependencies are installed at the point of install.

Therefore building a full dependency tree just from the metadata in the manifest is potentially not fully feasible. With much of it being implicitly inferred with properties like PowerShellVersion being set.

It may therefore be determined as aspirational in adding this data, but of minor value, unless in future all this additional data is added somehow. Particularly as there may be instances where bundled modules across differing PowerShell versions as well as Windows Client/Server versions may instead be better served by community modules, especially those that are xPlat friendly.

This is such a huge area to consider I only mention it so that it is thought about. I however feel this is a post implementation topic as this likely may take years to correct, even if it is correctable via the manifest data in a sensible way at all.

### Consideration 12

This issue is one of the hardest problems facing package management systems in use today, whether that be the PowerShell Gallery or other ecosystems like npm pip nuget or likes of winget, chocolatey, apt etc

As such I hope this RFC and some of the proposed changes, can also be taken on board to these other ecosystems too and over time help improve the processes around finding out about either a deprecated package or seemingly zombie package and the recommendations for replacements, without digging through blogs, tweets/skeets, conference talks etc

Whilst we will need to be careful that this doesn't also bring lots of `bad actor` reports, by having this we should be able in time to build a useful enough set of data with this to build additional reporting or scoring mechanisms around the reports that this functionality will provide.

This however is a consideration for perhaps 2 years post initial implementation, where by that time, we should have enough useful data (I hope) to analyse the true effectiveness of being able to add this data.

### Consideration 13

We perhaps should have some recommended standard messages for use in the *Reason Fields and either disallow them to be free-text fields when published to places like the PowerShell Gallery or have this flag in some way as part of the publishing process, whether in PSResourceGet or via other means like via server side analysis that happens during or post the publishing process. 

Doing so would further help against the `bad actor` reports with stuff like `This Module is so bad the Author should not be allowed to touch a computer or write code ever again.`

This may be something that as part of the Implementation has a number of starter messages and this list can be easily updated as well via future releases.

However for internally, non-public facing repositories this may make sense to not be so restrictive as to be able to point to internal KB articles etc.

## Required Help

As detailed in Consideration 6 and due to the wide ranging impact this will have, there will be a need to co-ordinate this work across all impacted areas, with most of this being able to be fleshed out as part of this RFC, bearing in mind all considerations listed above which I believe are all completely in the open.

This can all be managed using a GitHub Project that sits under the PowerShell Organisation or under a designated project lead's GitHub account as long as it's made publicly visible. This can be of use in large projects and helps make work more visible.

Use of GitHub Projects allow use of Tables, Backlogs and Monthly/Quarterly Roadmap views. Which may be of use, however it may also be more effort than it's worth for this particular RFC & is brought up to inform readers of this RFC to investigate in your development processes.

The PowerShell Team have already been making use of Projects in a few places already. The key benefit of using one with this project is that they allow cross repository tracking. 

Therefore, it may be worth during Implementation Stage for a community champion for this change to regularly review this work with whoever ends tasked to implement this, whether in the Team or from the community. Whilst this need not be myself, this suggestion is left in  for debate, as whether or not this is both suitable and scalable in future for the team to take as potentially new extended process. 