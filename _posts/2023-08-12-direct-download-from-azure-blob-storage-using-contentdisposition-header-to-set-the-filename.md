---
published: true
title: "Direct download from Azure Blob storage using Content-Disposition header to set the filename"
tags:
 - Azure
header:
  teaser: 'https://mindbyte.nl/images/contentdisposition.png'
---

For an application I m building, I store attachments on Azure in blob storage. This service is a relatively cheap and scalable solution for files that need to be retrieved again by others.

I have set the container to be a public one, and by generating random file names, I have some obfuscation so that iterating over them becomes complicated, but the URL is still sharable. These GUIDs also allow me to avoid name collisions as it is user input where I have no saying in what they upload.

But when you now want to retrieve the file, you get that random file name and not the original one it was uploaded with.

I can download the file to my server first and then return it with a correct name, but this will bring the additional IO to my system. I rather have people directly hit the Azure infrastructure.

Luckily you can set the `content-disposition` header. This header tells the client what to do with the file, like showing it inline or downloading it, and what the filename needs to be. This property is in there for some years, so you can set this directly using the SDK when you create a file:

```csharp
 var blobName = $"{cId}/{rId}/{Guid.NewGuid():N}.{extension}";
 
 BlobClient blobClient = containerClient.GetBlobClient(blobName);

 // Upload a byte[] to a blob
 await blobClient.UploadAsync(
 new MemoryStream(attachmentContent),
 new BlobHttpHeaders
 {
 ContentType = attachmentType,
 ContentDisposition = $"attachment; filename=\"{attachmentName}\""
 });

 string downloadUrl = blobClient.Uri.AbsoluteUri;

```

However, when I now used the public URL, I still got the long GUID-like name back, and the `content-disposition` header was missing. It appears that you need to specify the `x-ms-version: 2019-12-12` header as well. This is a bit annoying to set on a GET request in a browser as you cannot specify any headers, so we need to force the storage account to always use this version. 

There is no easy dropdown in the portal, and as it is a once-off action, you can use the Cloud Shell (with Powershell) to do this:

```powershell
$ctx = New-AzureStorageContext -StorageAccountName name-of-storageaccount -StorageAccountKey your-access-key
Update-AzureStorageServiceProperty -ServiceType Blob -DefaultServiceVersion 2019-12-12 -Context $ctx
```

When you now fetch the file using the public url, you get the header included and as such the correct filename. But it took me some while to figure out that this is not the default behaviour of the storage account, so make sure to check the `DefaultServiceVersion`. 