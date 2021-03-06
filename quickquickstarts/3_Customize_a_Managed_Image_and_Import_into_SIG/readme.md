# Create a Custom Image, from an Existing Custom Managed Image, then Distribute and Version over Multiple Regions

This article is to show you how you can use an existing custom [Managed Image](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/capture-image-resource) using the Azure VM Image Builder, and then use the Azure [Shared Image Gallery](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/shared-image-galleries).

To use this Quick Quickstarts, this can all be done using the Azure [Cloudshell from the Portal](https://azure.microsoft.com/en-us/features/cloud-shell/). Simply copy and paste the code from here, at a miniumum, just update the **subscriptionID** variable below.


## Step 0 : Enable Prereqs

Happy Image Building!!!

### Register for Image Builder / VM / Storage Features
```bash
az feature register --namespace Microsoft.VirtualMachineImages --name VirtualMachineTemplatePreview

az feature show --namespace Microsoft.VirtualMachineImages --name VirtualMachineTemplatePreview | grep state

# register and enable for shared image gallery
az feature register --namespace Microsoft.Compute --name GalleryPreview

# wait until it says registered

# check you are registered for the providers

az provider show -n Microsoft.VirtualMachineImages | grep registrationState
az provider show -n Microsoft.Storage | grep registrationState
az provider show -n Microsoft.Compute | grep registrationState
az provider show -n Microsoft.KeyVault | grep registrationState
```

If they do not saw registered, run the commented out code below.
```bash
## az provider register -n Microsoft.VirtualMachineImages
## az provider register -n Microsoft.Storage
## az provider register -n Microsoft.Compute
## az provider register -n Microsoft.KeyVault


```
Lastly ensure you have an existing Custom Managed Image, if you dont, you need to create one, you can go to Quick QuickStart 0_Creating_a_Custom_Linux_Managed_Image. 

## Step 1 : Set Permissions & Create Shared Image Gallery (SIG)

For Preview AIB will only support creating custom images in the same Resource Group as the source custom managed image. For example, if your existing managed custom images resides in RG1, then you must make sure the sigResourceGroup variable below is set to RG1. In the quick start below, we will create a SIG in that RG.

This Quick Start assumes you have completed 10_Creating_a_Custom_Linux_Managed_Image, have the resultant managed image and therefore the variables below, will be preset to those variable names, for continuity, but you can always update them yourself.

```bash
# set your environment variables here!!!!

# Create SIG  resource group
sigResourceGroup=aibmdi

# location of SIG (see possible locations in main docs)
location=westus2

# additional region to replication image to
additionalregion=eastus

# your subscription
# get the current subID : 'az account show | grep id'
subscriptionID=<INSERT YOUR SUBSCRIPTION ID HERE>

# source image name
srcImageName=aibCustomLinuxImg01

# name of the shared image gallery, e.g. myCorpGallery
sigName=my10thSIG

# name of the image definition to be created, e.g. ProdImages
imageDefName=ubuntu1804images

# name of the image definition to be created, e.g. ProdImages
runOutputName=sigRo

# create SIG
az sig create \
    -g $sigResourceGroup \
    --gallery-name $sigName

# create SIG image definition
##$sigName \
az sig image-definition create \
   -g $sigResourceGroup \
   --gallery-name $sigName \
   --gallery-image-definition $imageDefName \
   --publisher corpIT \
   --offer myOffer \
   --sku 18.04-LTS \
   --os-type Linux

```

### Assign AIB SPN Permissions to distribute a Managed Image or Shared Image 
```bash
# download preconfigured example
curl https://raw.githubusercontent.com/danielsollondon/azvmimagebuilder/master/solutions/12_Creating_AIB_Security_Roles/aibRoleImageCreation.json -o aibRoleImageCreation.json

# update the definition
sed -i -e "s/<subscriptionID>/$subscriptionID/g" aibRoleImageCreation.json
sed -i -e "s/<rgName>/$sigResourceGroup/g" aibRoleImageCreation.json

# create role definitions
az role definition create --role-definition ./aibRoleImageCreation.json

# grant role definition to the AIB SPN
az role assignment create \
    --assignee cf32a0cc-373c-47c9-9156-0db11f6a6dfc \
    --role "Azure Image Builder Service Image Creation Role" \
    --scope /subscriptions/$subscriptionID/resourceGroups/$sigResourceGroup

```

## Step 2 : Modify HelloImage Example

```bash
# download the example and configure it with your vars

curl https://raw.githubusercontent.com/danielsollondon/azvmimagebuilder/master/quickquickstarts/11_Creating_a_Custom_Image_from_Custom_Managed_Image/helloImageTemplateforReCustomization.json -o helloImageTemplateforReCustomization.json

sed -i -e "s/<subscriptionID>/$subscriptionID/g" helloImageTemplateforReCustomization.json
sed -i -e "s/<rgName>/$sigResourceGroup/g" helloImageTemplateforReCustomization.json
sed -i -e "s/<imageDefName>/$imageDefName/g" helloImageTemplateforReCustomization.json
sed -i -e "s/<sharedImageGalName>/$sigName/g" helloImageTemplateforReCustomization.json
sed -i -e "s/<srcImageName>/$srcImageName/g" helloImageTemplateforReCustomization.json


sed -i -e "s/<region1>/$location/g" helloImageTemplateforReCustomization.json
sed -i -e "s/<region2>/$additionalregion/g" helloImageTemplateforReCustomization.json
sed -i -e "s/<runOutputName>/$runOutputName/g" helloImageTemplateforReCustomization.json

```

## Step 3 : Create the Image

```bash
# submit the image confiuration to the VM Image Builder Service

az resource create \
    --resource-group $sigResourceGroup \
    --properties @helloImageTemplateforReCustomization.json \
    --is-full-object \
    --resource-type Microsoft.VirtualMachineImages/imageTemplates \
    -n ImgTemplateforCustomManImg01


# start the image build

az resource invoke-action \
     --resource-group $sigResourceGroup \
     --resource-type  Microsoft.VirtualMachineImages/imageTemplates \
     -n ImgTemplateforCustomManImg01 \
     --action Run 

# wait minimum of 20mins (this includes replication time to both regions)
```


## Step 4 : Create the VM

```bash
az vm create \
  --resource-group $sigResourceGroup \
  --name aibImgRcVm01 \
  --admin-username aibuser \
  --location $location \
  --image "/subscriptions/$subscriptionID/resourceGroups/$sigResourceGroup/providers/Microsoft.Compute/galleries/$sigName/images/$imageDefName/versions/latest" \
  --generate-ssh-keys

# and login...

ssh aibuser@<pubIp>

You should see the image was customized with a Message of the Day as soon as your SSH connection is established!

*******************************************************
**            This VM was built from the:            **
...

```

## Clean Up
```bash
# BEWARE : This is DELETING the Image created for you, be sure this is what you want!!!

# delete AIB Template
az resource delete \
    --resource-group $sigResourceGroup \
    --resource-type Microsoft.VirtualMachineImages/imageTemplates \
    -n ImgTemplateforCustomManImg01

# delete role assignment
az role assignment delete \
    --assignee cf32a0cc-373c-47c9-9156-0db11f6a6dfc \
    --role "Azure Image Builder Service Image Creation Role" \
    --scope /subscriptions/$subscriptionID/resourceGroups/$sigResourceGroup

az role definition delete --name "Azure Image Builder Service Image Creation Role"


# get image version created by AIB, this always starts with 0.*
sigDefImgVersion=$(az sig image-version list \
   -g $sigResourceGroup \
   --gallery-name $sigName \
   --gallery-image-definition $imageDefName \
   --subscription $subscriptionID --query [].'name' -o json | grep 0. | tr -d '"')

# delete image version
az sig image-version delete \
   -g $sigResourceGroup \
   --gallery-image-version $sigDefImgVersion \
   --gallery-name $sigName \
   --gallery-image-definition $imageDefName \
   --subscription $subscriptionID

# delete image definition
az sig image-definition delete \
   -g $sigResourceGroup \
   --gallery-name $sigName \
   --gallery-image-definition $imageDefName \
   --subscription $subscriptionID

# delete SIG
az sig delete -r $sigName -g $sigResourceGroup

# delete RG
az group delete -n $sigResourceGroup -y
```

## Next Steps
If you loved or hated Image Builder, please go to next steps to leave feedback, contact dev team, more documentation, or try more examples [here](../quickquickstarts/nextSteps.md)]

