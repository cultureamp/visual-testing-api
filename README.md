# visual-testing-api

This is an open source project which is a copy of visual-droid api. The code is provided as is without any guarantee or warranties. We have no active maintainers for this repository.

If you have questions, ask the Front End Capability Team at Culture Amp (#team_front_end in #camp_amplify)

Original documentation is below

# visual-droid
Visual-droid is a serverless REST Api that stores images and does image comparison. It uses [Looks Same](https://github.com/gemini-testing/looks-same) algo to compare the images.

<!-- TOC depthFrom:2 -->

- [Getting Started](#getting-started)
    - [Clone](#clone)
    - [Local Setup](#local-setup)
    - [Run unit test](#run-unit-test)
        - [Locally](#locally)
        - [Docker](#docker)
    - [Run integration test](#run-integration-test)
        - [Locally](#locally-1)
        - [Docker](#docker-1)
    - [Run linting](#run-linting)
        - [Locally](#locally-2)
        - [Docker](#docker-2)
- [Deploying aws stack](#deploying-aws-stack)
- [Schema Documentation](#schema-documentation)
    - [saveAndCompareImage](#saveandcompareimage)
        - [Response](#response)
            - [New Images](#new-images)
            - [Images Match](#images-match)
            - [Images Mismatch](#images-mismatch)
    - [saveImage](#saveimage)
        - [Save Image (without triggering compare)](#save-image-without-triggering-compare)
        - [Save Image (with triggering compare)](#save-image-with-triggering-compare)
    - [listOfNewImages](#listofnewimages)
    - [listOfImageDiff](#listofimagediff)
    - [setTestStatus](#setteststatus)
    - [getTestStatus](#getteststatus)
    - [deleteFolder](#deletefolder)
    - [imagesProcessed](#imagesprocessed)
    - [retryCompareImage](#retrycompareimage)
- [Architecture Flow](#architecture-flow)
    - [saveAndCompareImage Flow](#saveandcompareimage-flow)
    - [saveImage Flow](#saveimage-flow)
    - [listOfNewImages Flow](#listofnewimages-flow)
    - [retryCompareImage Flow](#retrycompareimage-flow)
    - [updateCode Flow](#updatecode-flow)
- [How to](#how-to)
    - [Create a stack](#create-a-stack)
        - [Pre requisites](#pre-requisites)
        - [Setup](#setup)
    - [Update the stack](#update-the-stack)
    - [Update lambda functions](#update-lambda-functions)
    - [Setup CI permissions](#setup-ci-permissions)
- [Typical work flow](#typical-work-flow)
- [Sample scripts](#sample-scripts)
    - [preTestRun.sh](#pretestrunsh)
    - [postTestRun.sh](#posttestrunsh)
    - [setTestStatus.sh](#setteststatussh)

<!-- /TOC -->

## Getting Started

### Clone
To clone this repo, run these commands
```
mkdir -p ~/code/cultureamp
git clone git@github.com:cultureamp/visual-droid.git ~/code/cultureamp/visual-droid
```

### Local Setup
```
npm install
```

### Run unit test

#### Locally
```
npm run test:unit:local
```

#### Docker
```
npm run test:unit:docker
```

### Run integration test

#### Locally
```
npm run test:integration:local
```

#### Docker
```
npm run test:integration:docker
```

### Run linting

#### Locally
```
npm run lint:local
```

#### Docker
```
npm run lint:docker
```

## Deploying aws stack
Read [How to](#how-to) section

## Schema Documentation
The App is made up of lambda functions which get accessed by an api gateway. The lambda functions save the images in an s3 bucket. All the calls to this api are authenticated using a username and password. It is advisable to keep these in a secure place. The username and password need to be shared between this app and the app making the requests.

To use the app, first standup the stack. Then run the integration tests to make sure the end points are working.

The app provides the following end points.

- [saveAndCompareImage](#saveandcompareimage)
- [saveImage](#saveimage)
- [listOfNewImages](#listofnewimages)
- [listOfImageDiff](#listofimagediff)
- [setTestStatus](#setteststatus)
- [getTestStatus](#getteststatus)
- [deleteFolder](#deletefolder)
- [imagesProcessed](#imagesProcessed)
- [retryCompareImage](#retryCompareImage)

All the end points would need a basic Authentication header (Specify your own username and password and base64encode it)
```
Basic dXNlcjpwYXNzd29yZA==
```

### saveAndCompareImage
This is an end Point for saving and image and doing comparison.

To use the end point you would need to make a POST request to this end point with the payload. Metadata object values are optional. You can also pass the options to the looks-same-logo. The app accepts the same options as defined here https://github.com/gemini-testing/looks-same

```
{
  project: {
    name: PROJECT_NAME,
    branchSha: COMMIT_SHA,
    compareSha: PARENT_BRANCH_SHA
  },
  image: {
    key: UNIQUE_NAME_FOR_THIS_IMAGE,
    base64EncodeString: BASE64_ENCODED_IMAGE,    
    looksSameAlgoOptions: { strict: true ,ignoreAntialiasing: false },
  },
  metadata: {
    imageName: ,
    width: '1',
    height: '2',
    os: '3',
    osVersion: '4',
    browser: '5',
    browserVersion: '6'
  }
}
```
A typical curl request looks like this:

```
curl -X POST \
  https://YOUR_API_URL/ENDPOINT/saveAndCompareImage \
  -H 'Authorization: Basic dXNlcjpwYXNzd29yZA==' \  
  -H 'Content-Type: application/json' \  
  -d '{
    "project": {
      "name": "temp",
      "branchSha": "branch",
      "compareSha": "master"
    },
    "image": {
      "key": "key",
      "base64EncodeString":"data:image/png;base64,iVBOw/..==",
      "looksSameAlgoOptions": { "strict": "true" ,"ignoreAntialiasing": "false" },
    },
    "metadata": {
      "imageName":"imageName",
      "width": "1",
      "height": "2",
      "os": "3",
      "osVersion": "4",
      "browser": "5",
      "browserVersion": "6"
    }
  }'
```

When we do image comparison we could end up in three scenarios.
- [New Images](#new-images) - Image we want to compare with does not exists. Sometimes we do end up having new screens or new tests which will create new image snapshots.
- [Images match](#images-match)  Images we are comparing match
- [Images mismatch](#images-mismatch)  Images we are comparing do not match

#### Response
##### New Images
If there are no images to compare to in the compareSha folder with the key specified then you will get back the url for the recently uploaded file and the response will look like the following and the url is valid for an hour. This time can be tweaked in the lambda function to your requirements.

```
{
    "upload": true,
    "urls": {
        "newUrl": "https://YOUR_S3BUCKET_URL/temp/branch/new/key?AWSAccessKeyId=ASIAIM5O18&Sig..."
    },
    "imagesMatch": false,
    "key": "key"
}
```

##### Images Match
```
{
    "upload": true,
    "imagesMatch": true,
    "key": "key"
}
```

##### Images Mismatch
If images don't match we will get back three url's. One for the image under test(branch), image we are comparing(compareTo) it to and differential image of the two images(diff)
```
{
  {
  "upload": true,
  "urls": {
      "diff": "https://YOUR_S3BUCKET_URL/...",
      "branch": "https://YOUR_S3BUCKET_URL/...",
      "compareTo": "https://YOUR_S3BUCKET_URL/..."
  },
  "imagesMatch": false,
  "key": "key"
}
```
 Mismatched images will look like the following with the diff part highlighted in red and everything else blurred

Original Image

![Original Image](images/readme/master_image.png)

Branch Image

![Branch Image](images/readme/branch_image.png)

Diff  Image

![Diff Image](images/readme/diff.png)


### saveImage
This is an end point for saving an image. You could either use this end point to save master images or use it to trigger the comparison as a background task.

To use the end point you would need to make a POST request to this end point with the payload

Metadata object values are optional.

#### Save Image (without triggering compare)
```
{
  project: {
    name: PROJECT_NAME,
    branchSha: COMMIT_SHA    
  },
  image: {
    key: UNIQUE_NAME_FOR_THIS_IMAGE
    base64EncodeString: BASE64_ENCODED_IMAGE
  },
  metadata: {
    imageName: ,
    width: '1',
    height: '2',
    os: '3',
    osVersion: '4',
    browser: '5',
    browserVersion: '6'
  }
}
```
A typical curl request looks like this:

```
curl -X POST \
  https://YOUR_API_URL/ENDPOINT/saveImage \
  -H 'Authorization: Basic dXNlcjpwYXNzd29yZA==' \  
  -H 'Content-Type: application/json' \  
  -d '{
    "project": {
      "name": "temp",
      "branchSha": "branch"      
    },
    "image": {
      "key": "key",
      "base64EncodeString":"data:image/png;base64,iVBOw/..=="
    },
    "metadata": {
      "imageName":"imageName",
      "width": "1",
      "height": "2",
      "os": "3",
      "osVersion": "4",
      "browser": "5",
      "browserVersion": "6"
    }
  }'
```

The response will look like this:
```
{
    "upload": true
}
```

#### Save Image (with triggering compare)
Sending the compareSha, ensures that a background process will do the comparison. You can also pass in the looksSameAlgoOptions, which will compare the images based on your preference.

```
{
  project: {
    name: PROJECT_NAME,
    branchSha: COMMIT_SHA,
    compareSha: PARENT_BRANCH_SHA    
  },
  image: {
    key: UNIQUE_NAME_FOR_THIS_IMAGE
    base64EncodeString: BASE64_ENCODED_IMAGE,
    looksSameAlgoOptions: { strict: true ,ignoreAntialiasing: false }
  },
  metadata: {
    imageName: ,
    width: '1',
    height: '2',
    os: '3',
    osVersion: '4',
    browser: '5',
    browserVersion: '6'
  }
}
```
A typical curl request looks like this:

```
curl -X POST \
  https://YOUR_API_URL/ENDPOINT/saveImage \
  -H 'Authorization: Basic dXNlcjpwYXNzd29yZA==' \  
  -H 'Content-Type: application/json' \  
  -d '{
    "project": {
      "name": "temp",
      "branchSha": "branch"      
    },
    "image": {
      "key": "key",
      "base64EncodeString":"data:image/png;base64,iVBOw/..=="
    },
    "metadata": {
      "imageName":"imageName",
      "width": "1",
      "height": "2",
      "os": "3",
      "osVersion": "4",
      "browser": "5",
      "browserVersion": "6"
    }
  }'
```

The response will look like this:
```
{
    "upload": true
}
```

### listOfNewImages
This is an end Point for retrieving url's of any new images that might have been generated as part of your saveAndCompareImage. You would probably use this end point if your tests have failed and you want to see if there were any new images generated.

To use the end point you would need to make a POST request to this end point with the payload
```
{
  "project": {
    "name": "temp",
    "branchSha": "branch"
  }
}
```
A typical curl request looks like this:

```
curl -X POST \
  https://YOUR_API_URL/ENDPOINT/listOfNewImages \
  -H 'Authorization: Basic dXNlcjpwYXNzd29yZA==' \  
  -H 'Content-Type: application/json' \  
  -d '{
    "project": {
      "name": "temp",
      "branchSha": "branch"      
    }
  }'
```

The response will look like this:
```
{
    "metadataFilters": {
        "browserversion": [
            "6",
            "60"
        ],
        "imagename": [
            "imageName",
            "googleimage"
        ],
        "width": [
            "1",
            "10"
        ],
        "browser": [
            "5",
            "50"
        ],
        "osversion": [
            "4",
            "40"
        ],
        "height": [
            "2",
            "20"
        ],
        "os": [
            "3",
            "30"
        ]
    },
    "images": [
        {
            "key": "key",
            "metadata": {
                "browserversion": "6",
                "imagename": "imageName",
                "width": "1",
                "browser": "5",
                "osversion": "4",
                "height": "2",
                "os": "3"
            },
            "imageUrl": "https://YOUR_S3BUCKET_URL/..",
            "thumbnailUrl":"https://YOUR_S3BUCKET_URL/.."
        },
        {
            "key": "key1",
            "metadata": {
                "browserversion": "60",
                "imagename": "googleimage",
                "width": "10",
                "browser": "50",
                "osversion": "40",
                "height": "20",
                "os": "30"
            },
            "imageUrl": "https://YOUR_S3BUCKET_URL/..",
            "thumbnailUrl": "https://YOUR_S3BUCKET_URL/.."
        }
    ]
}
```
This response has two new images.
The response will have all the metadata filters grouped by each metadata. It will also return the urls of the image and its thumbnails(lower resolution image)


### listOfImageDiff
This is an end Point for retrieving url's of any images that were different that might have been generated as part of your saveAndCompareImage. You would probably use this end point if your tests have failed and you want to see if there were any diff images generated.

To use the end point you would need to make a POST request to this end point with the payload
```
{
  "project": {
    "name": "temp",
    "branchSha": "branch",
    "compareSha": "master"
  }
}
```
A typical curl request looks like this:

```
curl -X POST \
  https://YOUR_API_URL/ENDPOINT/listOfImageDiff \
  -H 'Authorization: Basic dXNlcjpwYXNzd29yZA==' \  
  -H 'Content-Type: application/json' \  
  -d '{
    "project": {
      "name": "temp",
      "branchSha": "branch",
      "compareSha": "master"      
    }
  }'
```

The response will look like this:
```
{
    "metadataFilters": {
        "browserversion": [
            "6"
        ],
        "imagename": [
            "imageName"
        ],
        "width": [
            "1"
        ],
        "browser": [
            "5"
        ],
        "osversion": [
            "4"
        ],
        "height": [
            "2"
        ],
        "os": [
            "3"
        ]
    },
    "images": [
        {
            "key": "key",
            "metadata": {
                "browserversion": "6",
                "imagename": "imageName",
                "width": "1",
                "browser": "5",
                "osversion": "4",
                "height": "2",
                "os": "3"
            },
            "urls": {
                "diff": "https://YOUR_S3BUCKET_URL/..",
                "branch": "https://YOUR_S3BUCKET_URL/..",
                "compareTo": "https://YOUR_S3BUCKET_URL/.."
            },
            "thumbnailUrls": {
              "diff": "https://YOUR_S3BUCKET_URL/..",
              "branch": "https://YOUR_S3BUCKET_URL/..",
              "compareTo": "https://YOUR_S3BUCKET_URL/.."
            }
        }
    ]
}
```
This response one diff image.
The response will have all the metadata filters grouped by each metadata. It will also return the urls of the images(diff/branch/comparedto) and its thumbnails(lower resolution image)


### setTestStatus
This is an end point for setting the test status. Sometimes you want to overwrite the test status because you do expect the tests to fail. You would ideally only set the status to pass as the fail status is automatically returned if there are new images or diff images.

To use the end point you would need to make a POST request to this end point with the payload
```
{
  project": {
    "name": "temp",
    "branchSha": "branch"
  },
  "testStatus": false
}
```
testStatus can be true or false

A typical curl request looks like this:

```
curl -X POST \
  https://YOUR_API_URL/ENDPOINT/setTestStatus \
  -H 'Authorization: Basic dXNlcjpwYXNzd29yZA==' \  
  -H 'Content-Type: application/json' \  
  -d '{
    "project": {
      "name": "temp",
      "branchSha": "branch"
    },
    "testStatus": false
  }'
```

The response will look like this:
```
{
    "statusUpdated": true,
    "setStatusTo": false
}
```

### getTestStatus
This is an end point for getting the test status. The test status can be returned pass or fail. The test status is pass if there are no new images and no diff images. If you have explicitly set the test status as true using the end point setTestStatus then irrespective of new or diff images this end point will return the test status as pass.

For cases where there are new images or diff images or the branch you are trying to get a status for does not exist the test status returned is fail

To use the end point you would need to make a POST request to this end point with the payload
```
{
  project": {
    "name": "temp",
    "branchSha": "branch"
  }  
}
```

A typical curl request looks like this:

```
curl -X POST \
  https://YOUR_API_URL/ENDPOINT/getTestStatus \
  -H 'Authorization: Basic dXNlcjpwYXNzd29yZA==' \  
  -H 'Content-Type: application/json' \  
  -d '{
    "project": {
      "name": "temp",
      "branchSha": "branch_non_existant"
    }    
  }'
```

The response will look like this:
```
{
    "pass": false
}
```


### deleteFolder
This is an end point for deleting a folder on the s3 bucket you no longer need. Ideally you will use this end point to delete the branch commit sha you are trying to test. Even though you would probably run the test for a new commit sha, but this end point ensures that you delete the end point if you end up reusing the same commit sha.

To use the end point you would need to make a POST request to this end point with the payload
```
{
  project": {
    "name": "temp",
    "branchSha": "branch"
  }  
}
```

A typical curl request looks like this:

```
curl -X POST \
  https://YOUR_API_URL/ENDPOINT/deleteFolder \
  -H 'Authorization: Basic dXNlcjpwYXNzd29yZA==' \  
  -H 'Content-Type: application/json' \  
  -d '{
    "project": {
      "name": "temp",
      "branchSha": "branch_non_existant"
    }    
  }'
```

The response will look like this:
```
{
    "emptyFolder": true
}
```

### imagesProcessed
This is an end point that returns the images compared status. You need to use this end point when you use the saveImage end point with compareSha.

To use the end point you would need to make a POST request to this end point with the payload
```
{
  project": {
    "name": "temp",
    "branchSha": "branch"
  }  
}
```

A typical curl request looks like this:

```
curl -X POST \
  https://YOUR_API_URL/ENDPOINT/deleteFolder \
  -H 'Authorization: Basic dXNlcjpwYXNzd29yZA==' \  
  -H 'Content-Type: application/json' \  
  -d '{
    "project": {
      "name": "temp",
      "branchSha": "branch_existant"
    }    
  }'
```

The response will look like this:
```
{
    "status": true
}
```
or
```
{
    "status": false,
    "progress": "4/20"
}
```

### retryCompareImage
This endpoint triggers the compare of the remaining images. You would use this endpoint in case the  imagesProcessed endpoint returns `status:false` and progress never changes.

To use the end point you would need to make a POST request to this end point with the payload
```
{
  project": {
    "name": "temp",
    "branchSha": "branch"
  }  
}
```

A typical curl request looks like this:

```
curl -X POST \
  https://YOUR_API_URL/ENDPOINT/deleteFolder \
  -H 'Authorization: Basic dXNlcjpwYXNzd29yZA==' \  
  -H 'Content-Type: application/json' \  
  -d '{
    "project": {
      "name": "temp",
      "branchSha": "branch"
    }    
  }'
```

The response will look like this:
```
{
    "retryProcessed": true
}
```

## Architecture Flow
The App is made up of few lambda functions, a sns topic, two s3 buckets and an Api Gateway
<img src="Architectural Diagram.png"/>

### saveAndCompareImage Flow
<img src="images/readme/saveAndCompareImage.png"/>

1. Api Request is serviced by the Visual Droid Api Gateway
2. Api Gateway calls the auth function that does the authentication. If successful it goes to step 3, other wise a 401 Unauthorized Error is returned.
3. saveAndCompareImage lambda function is called.
4. Function saves an image in the s3 bucket and also retrieves an image if a comparative image is found.
5. Saving of the image triggers a notification to be added to sns.
6.  * createThumbnail lambda gets triggered 
    * compareImage thumbnail gets triggered (does not process any images, see code) 
7. Thumbnail for the new image gets created.

### saveImage Flow
<img src="images/readme/saveImage.png"/>

1. Api Request is serviced by the Visual Droid Api Gateway
2. Api Gateway calls the auth function that does the authentication. If successful it goes to step 3, other wise a 401 Unauthorized Error is returned.
3. saveImage lambda function is called.
4. Function saves an image in the s3 bucket with some metadata (see code).
5. Saving of the image triggers a notification to be added to sns.
6.  * createThumbnail lambda gets triggered 
    * compareImage thumbnail gets triggered 
7. Thumbnail for the new image gets created. Also an image comparison status file(to keep track of images processed). It can also create an new Image(if there was no image to compare to), or a diff image if the images did not match. 

### listOfNewImages Flow
<img src="images/readme/listOfNewImages.png"/>

1. Api Request is serviced by the Visual Droid Api Gateway
2. Api Gateway calls the auth function that does the authentication. If successful it goes to step 3, other wise a 401 Unauthorized Error is returned.
3. listOfNewImages lambda function is called.
4. Function retrieves the list of new images if any.

This is true for the following endpoints
- listOfImageDiff
- setTestStatus
- getTestStatus
- deleteFolder
- imagesProcessed
- listImages


### retryCompareImage Flow
<img src="images/readme/retryCompareImage.png"/>

1. Api Request is serviced by the Visual Droid Api Gateway
2. Api Gateway calls the auth function that does the authentication. If successful it goes to step 3, other wise a 401 Unauthorized Error is returned.
3. retryCompareImage lambda function is called.
4. Function copies the existing image in the current location. 
5. Copying images works as if new images has been added and creates a notification for each of those images
6.  * createThumbnail lambda gets triggered 
    * compareImage thumbnail gets triggered 
7. Thumbnail for the new image gets created. Also an image comparison status file(to keep track of images processed). It can also create an new Image(if there was no image to compare to), or a diff image if the images did not match. 


### updateCode Flow
<img src="images/readme/updateCode.png"/>

1. Add the zip file of this project (look at the [uploadCode.sh](aws/cloudFormation/uploadCode.sh)) to the visual-droid-source-code bucket.
2. This will trigger a notification to the updateVisualDroidLambdaCode function.
3. The function then retrieves the list of lambdas that it needs to update and updates those.

## How to

### Create a stack

#### Pre requisites
- The setup expects you to have two aws accounts. One is for testing and other it for running against your CI environment.
- We need certain environment variables to be available to us so that we can create the stack. The list of environment variables are in [this](aws/cloudFormation/addEnvParameters.sh) file. 

#### Setup
1. Log into your aws account
2. Run the [addEnvParameters](aws/cloudFormation/addEnvParameters.sh) script
3. Run the [createStack](aws/cloudFormation/createStack.sh) script

### Update the stack
Stack can be updated in two ways
- Update metadata (roles/permission/tags etc.)
    1. Log into your aws account
    2. Run the [updateStack](aws/cloudFormation/updateStack.sh) script

- Update stack structure (eg. Adding/removing endpoints)
    1. Update code 
        - Add/remove the lambda function in the src/lambdas folder
        - Import/delete the new function in the lambdas.js file
        - Add/remove the endpoint configuration in the swagger.yml
        - Add/remove the endpoint configuration in the serverless.template.yml
    2. Log into your aws account
    3. Run the [updateStack](aws/cloudFormation/updateStack.sh) script

### Update lambda functions
1. Log into your aws account
2. Run the [uploadCode](aws/cloudFormation/uploadCode.sh) script

### Setup CI permissions
In order for build agents to be able to run your tests, they need permission to upload code into the s3 bucket. This means we need to give them permissions to do so. The roles for this is set in the [createCiDeploymentRoles](aws/cloudFormation/createCiDeploymentRoles.sh) script.

1. Log into your aws account
2. Run the [createCiDeploymentRoles](aws/cloudFormation/createCiDeploymentRoles.sh) script


## Typical work flow

You should ideally run your tests(unit/regression etc) for each commit. Which means your visual comparison test will also run on each commit. This commit will become the BRANCH_SHA. To do the comparison we would need a branch to compare it to. This will be the COMPARE_SHA. We have decided to do a comparison with the parent branch this branch was branched off of. This makes it easy to make sure that any changes introduced by anyone else does not affect your changes. The job of taking snapshots and sending these snapshots resides with the testing app you are using. This framework is agnostic of what language or tools you use to take snap shots. As long as you can take snapshots and send it this app, will do the comparison for you.

Compare the branch to the commit sha you branch off from
<img src="images/readme/before_master_merge.png"/>

After merging master into your branch you will compare to the last merge commit you just merged into your branch
<img src="images/readme/after_master_merge.png"/>

Here are some helpful commands to get the variables setup for your tests


```
BRANCH_NAME=$(git rev-parse --abbrev-ref HEAD)
BRANCH_SHA=$(git rev-parse HEAD)
COMPARE_SHA=$(git merge-base origin/master "$BRANCH_SHA")
```
The test flow will be different for your master branch vs feature branches. On the master branch, when you are running your tests, you don't need to compare the images, you just need to save them. Any branch that gets branched off of this master will have all the images it need to compare to.

1. Delete the folder with the BRANCH_SHA. This ensures that the new run is clean
2.  * Running tests for a branch
      * Do a saveAndCompareImage. This is a synchronous call, which will save and compare the image.
      * Do a saveImage (payload needs to have compareSha). This will save the image and do the compare asynchronously as background task.
    * Running tests for a master
      * Do a saveImage

  When running in CI, you do not want the tests to fail even if the image comparison would have failed. Visual tests could fail because of three reasons
    * New images because of new test
    * Images are are supposed to be different because of visual changes you made
    * Genuine image Mismatch
  The test status is reported by calling the getStatus endpoint  

3. If you did a saveImage (payload with compareSha), you will have to check the comparison status of images by using the endpoint imagesProcessed
4. To make sure if the tests have passed, do check for the test status using the end point getTestStatus. If there were new or diff images then you will get the status as fail. You will not need to do a test status check on the master branch as we are not doing a comparison on master, but instead just saving images.
4. If the test status was fail you can retrieve the new images or diff images by hitting the endpoints listOfImageDiff and listOfNewImages. Currently we don't have an endpoint to tell us why the test failed. Did it fail because of new or diff images. You will need to access both endpoints to determine that.
5. If you look at the images and verify that the failures are expected, then you can call the setTestStatus end point and set the test status as true.
6. You can re run the step which reported the test status as failed and it will now report as the test passed.

## Sample scripts

These scripts will help you understand the flow and also get you off the ground quickly. They are used primarily in CI's context of the project you need to do visual tests for. These scripts are not to be used in context of this repo.

### preTestRun.sh
Run [this](sampleScripts/preTestRun.sh) script on branches before the visual tests have started running. It checks if the parent branch has finished running its tests, before you proceed to start doing visual tests.

### postTestRun.sh
Run [this](sampleScripts/postTestRun.sh) script on branches after the visual tests have finished running. It will check if the images have all been compared and what is the status of that comparison.

### setTestStatus.sh
Run [this](sampleScripts/setTestStatus.sh) script only on master as there is no other script running to update test status.
