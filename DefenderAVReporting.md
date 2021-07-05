# Microsoft Defender for Endpoint

**Blog about Defender AV reporting**

This blog is about getting Windows client Defender Antivirus detailed status information from different sources as part of Microsoft 365.
Regulary is get asked the question how to centraly report on Defender AV status. There are several ways to retrieve Defender AV status information and not every method is as abvious.
With this blog post i wanted to provide some insights and clarity on how to get the data out of M365 for reporting purposes.

Hopefully you will find it usefull!


## Exporting Defender AV reports from Microsoft Endpoint Manager by using the GraphAPI ##

Microsoft Endpoint Manager provides different reports accesible via the Microsoft Endpoint Manager admin center.
Reporting with Defender AV status can be found in the Reports pane. 
_Reports -> Microsoft Defender Antivirus -> Reports -> Antivirus agent status -> Generate report_
Generating this report for the first time can take some time and eventually results are visible and we can manualy export the details as a .csv via the export button.

To automate this process we can initiated the export of this or other reports by the usage of the GraphAPI. 
Exporting can be acomplised by the usage of several HTTP POST and GETE commands to the graph API endpoint.

-   First we need to send a HTTP POST with detail information about the report we want to generate.
    In the HTTP POST body we need to provide the key reportname: `"reportName"="DefenderAgents"`
    We can send this HTTP POST request by the usage of powershell after authentication to the GRAPH API.
    Example powershell code:
    ```
    $JSON = @{
        "reportName"="DefenderAgents"
    } | ConvertTo-Json
    
        try {
    
            $uri = "https://graph.microsoft.com/beta/deviceManagement/reports/exportJobs"
            Write-Verbose $uri
            $id = (Invoke-RestMethod -Uri $uri -Headers $authToken -Method POST -Body $JSON).id
            Write-Host $id
        }
    
        catch {
    
        $ex = $_.Exception
        $errorResponse = $ex.Response.GetResponseStream()
        $reader = New-Object System.IO.StreamReader($errorResponse)
        $reader.BaseStream.Position = 0
        $reader.DiscardBufferedData()
        $responseBody = $reader.ReadToEnd();
        Write-Host "Response content:`n$responseBody" -f Red
        Write-Error "Request to $Uri failed with HTTP Status $($ex.Response.StatusCode) $($ex.Response.StatusDescription)"
        write-host
        break
    
        }
    ```
    In above powershell example we create a JSON object with the report type and send it as part of a HTTP POST.
-   As a reponse on above HTTP POST we will recieve a HTTP response with a ID this id we can use to initiate the export by the usage of a HTTP GET.
    We can send a HTTP GET request to the reports/exportJobs endpoint including the ID example: `https://graph.microsoft.com/beta/deviceManagement/reports/exportJobs('ID')`
    Example powershell code:
    ```
      try {
    
                $uri = "https://graph.microsoft.com/beta/deviceManagement/reports/exportJobs('$id')"
                Write-Verbose $uri
                $statusInfo = (Invoke-RestMethod -Uri $uri -Headers $authToken -Method GET)

                $status = $statusInfo.Status
                Write-Host $status
            }
        
            catch {
        
            $ex = $_.Exception
            $errorResponse = $ex.Response.GetResponseStream()
            $reader = New-Object System.IO.StreamReader($errorResponse)
            $reader.BaseStream.Position = 0
            $reader.DiscardBufferedData()
            $responseBody = $reader.ReadToEnd();
            Write-Host "Response content:`n$responseBody" -f Red
            Write-Error "Request to $Uri failed with HTTP Status $($ex.Response.StatusCode) $($ex.Response.StatusDescription)"
            write-host
            break
        
            }
        ```

-   The first reponse we will get back as a result of this HTTP GET is a inProgress state. The GET reponse above initate the export withs will take some time depending on the data.
    We can send several successive HTTP GET requests until we recieve the status completed. We can use for example a While loop in powershell. For a full PS code example including interactive authentication: LINK.

-   As part of the same HTTP GET Reponse with status completed we will recieve a URL address. This URL address can be used to download the exported .CSV in .zip format.
    We can again use powershell to download the .zip to a location on your local device in this example $ExportLocation. 
    Example powershell code:
    ```
     if($status -eq "completed"){
            Write-Host "completed";
           

            Write-Host $statusInfo.url
            Write-Host "Downloading... .csv"

            Invoke-WebRequest -Uri $statusInfo.url -OutFile $ExportLocation

        }
    ```
   
    The full powershell example can be download from: URLGIT
    

## Exporting Defender AV device status from Microsoft Endpoint Manager by using the GraphAPI ##

A other way how to retrieve Defender AV device status information is as part of device information that can be retrieved via the GRAPH API.
WindowsProtectionStatus is a entity of information as part of a managed device object. the WindowsProtectionStatus information can manually retrieved from the Graph API by requesting device information of a individual device by the usage of the unique device management ID and by the usage of the windowsProtectionState in the url. 

The HTTP GET request will for example look like: `https://graph.microsoft.com/beta/deviceManagement/managedDevices/UniqueManagementDeviceID/windowsProtectionState`
To retrieve this information from multiple devices you can use PowerShell to interative retrieve the UniqueManagementDeviceIDs and request the WindowsProtectionState.
The Example powershell can be downloaded from: URLGIT

A HTTP GET request reponse of a single device will look like:
```
{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#deviceManagement/managedDevices('MEMDeviceID')/windowsProtectionState/$entity",
    "id": "MEMDeviceID",
    "malwareProtectionEnabled": true,
    "deviceState": "clean",
    "realTimeProtectionEnabled": true,
    "networkInspectionSystemEnabled": true,
    "quickScanOverdue": false,
    "fullScanOverdue": false,
    "signatureUpdateOverdue": false,
    "rebootRequired": false,
    "fullScanRequired": false,
    "engineVersion": "1.1.17300.4",
    "signatureVersion": "1.321.492.0",
    "antiMalwareVersion": "4.18.2006.10",
    "lastQuickScanDateTime": null,
    "lastFullScanDateTime": null,
    "lastQuickScanSignatureVersion": "0.0.0.0",
    "lastFullScanSignatureVersion": "0.0.0.0",
    "lastReportedDateTime": "2020-08-03T09:43:14.7468187Z",
    "productStatus": null,
    "isVirtualMachine": null,
    "tamperProtectionEnabled": null
}
```
## What about retrieving the data from PowerBI ?##
At this moment of writing we can't unfortunate not natively connect to the DeviceManagement GRAPH API endpoints to retrieve data directly from PowerBI.
There are some workarrounds avliable to accomplise this think about:

-   Using Azure Automations and non-interactive auth by the usage of a Azure Service Principal to retrieve the data and save the data on blob storage.
    Via PowerBI we can connect to BLOB storage to create reports
-   Using Azure Service Principal(App Registration and Secreat) to directly authentication by the usage of PowerBI advanced Query. Major drawback concern here is how to secure the     Secret used in PowerBI Advanced Query. 


**Whats next? retrieving Defender AV status from Microsoft Defender for Endpoint**

