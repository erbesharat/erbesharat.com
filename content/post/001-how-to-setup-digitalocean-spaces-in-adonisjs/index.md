---
title: How to setup DigitalOcean Spaces in Adonis JS
description: A quick walkthrough on how to setup DigitalOcean Spaces using Adonis JS and its S3 driver
slug: how-to-setup-digitalocean-spaces-in-adonisjs
date: 2023-03-06 00:00:00+0000
image: cover.png
categories:
    - Adonis JS
    - DigitalOcean
    - Backend
tags:
    - Adonis JS
    - DigitalOcean
    - Backend
    - S3
    - Cloud Storage
    - Spaces
    - CDN
draft: false
---

## Introduction
The reason you'll want to use DigitalOcean Spaces is because it's a cheap and easy way to store your application's assets on cloud. It's also a great way to offload your server from having to serve files. You can also use a CDN to serve your files from Spaces, which will make your content load faster.

## Prerequisites
- An [Adonis JS](https://adonisjs.com/) application
- A [DigitalOcean](https://www.digitalocean.com/) account

## DigitalOcean Spaces Setup
1. Login to your DigitalOcean account and create a new Space. You can click [here](https://cloud.digitalocean.com/spaces/new) to create a new Space through the DigitalOcean dashboard.

2. Based on your needs, you can choose to create a Space in a specific region. You can also choose to enable CDN for your Space. You can read more about the different options [here](https://docs.digitalocean.com/products/spaces/how-to/create/#create-a-space).

3. Once you've created your Space, you'll need to create an Spaces access key. You can click [here](https://cloud.digitalocean.com/account/api/spaces) to go to the API section of your DigitalOcean account.

### Pulumi Infrastructure as Code
If you're using [Pulumi](https://www.pulumi.com/) to manage your infrastructure, you can use the following code to create a Space.

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as digitalocean from "@pulumi/digitalocean";

const myDOSpace = new digitalocean.SpacesBucket("myDOSpace", {
    corsRules: [
        {
            allowedHeaders: ["*"],
            allowedMethods: ["GET"],
            allowedOrigins: ["*"],
            maxAgeSeconds: 3000,
        },
        {
            allowedHeaders: ["*"],
            allowedMethods: [
                "PUT",
                "POST",
                "DELETE",
            ],
            allowedOrigins: ["https://www.erbesharat.com"],
            maxAgeSeconds: 3000,
        },
    ],
    region: "ams3",
});
```

## Adonis JS Setup
We're going to use [Adonis JS Drive](https://docs.adonisjs.com/guides/drive) to setup our Spaces driver. Adonis JS Drive is a official package that allows you to use different drivers to store your application's assets.

### Install Adonis JS S3 Driver
1. Use the following command to install the Adonis JS S3 driver:

```bash
npm i @adonisjs/drive-s3
```

2. Then use the `ace` command to configure the driver:

```bash
node ace configure @adonisjs/drive-s3
```

3. Update the `.env` file with the following values:

```bash
DRIVE_DISK=s3
S3_KEY=YOUR_SPACES_KEY
S3_SECRET=YOUR_SPACES_SECRET
S3_BUCKET=YOUR_SPACES_NAME
S3_REGION=YOUR_SPACES_REGION
S3_ENDPOINT=YOUR_SPACES_ENDPOINT
```

4. Then update the `config/drive.ts` file with the following values:

```typescript
import { driveConfig } from '@adonisjs/core/build/config'
import Application from '@ioc:Adonis/Core/Application'
import Env from '@ioc:Adonis/Core/Env'

export default driveConfig({
  disk: Env.get('DRIVE_DISK'),

  disks: {
    local: {
      driver: 'local',
      visibility: 'public',
      root: Application.tmpPath('uploads'),
      serveFiles: true,
      basePath: '/uploads',
    },
    s3: {
      driver: 's3',
      key: Env.get('S3_KEY'),
      secret: Env.get('S3_SECRET'),
      region: Env.get('S3_REGION'),
      bucket: Env.get('S3_BUCKET'),
      endpoint: Env.get('S3_ENDPOINT'),
    },
  },
})
```

## Uploading Files to Spaces
1. In the `app/Validators` directory, create a new file called `UploadFileValidator.ts` with the following code:

```typescript
import type { HttpContextContract } from '@ioc:Adonis/Core/HttpContext'
import { CustomMessages, schema } from '@ioc:Adonis/Core/Validator'

export default class UploadFileValidator {
  constructor(protected ctx: HttpContextContract) {}

  public schema = schema.create({
    file: schema.file({
      size: '5gb',
      extnames: [
        'jpg',
        'png',
        ...
        'OGG',
        'FLAC',
      ],
    }),
    type: schema.enum(['avatar', 'cover', 'thumbnail', 'video', 'audio']),
  })

  public messages: CustomMessages = {}
}
```

This is because we'll need a custom validator to validate the file before we upload it to Spaces. You can read more about Adonis JS validators [here](https://docs.adonisjs.com/guides/validator/introduction).

2. Create a new Controller called `FilesController` with the following code:

```typescript
export default class FilesController {
  public async create({ auth, request }: HttpContextContract) {
    // Only allow authenticated users to upload files
    const user = auth.use('api').user!

    // Use a custom validator to validate the file
    const payload = await request.validate(UploadFileValidator)

    // Generate a unique ID for the file and a directory path based on the file type and user ID
    const fileUID = nanoid()
    const dirPath = join(payload.type, user.uid, nanoid())

    // Upload the file to Spaces
    await payload.file.moveToDisk(
      `./${dirPath}`,
      {
        visibility: Env.get('DRIVE_DISK') === 'local' ? 'public' : 'private',
      },
      Env.get('DRIVE_DISK'),
    )

    return {
        file: payload.file,
    }
  }
}
```

If you're not familiar with Adonis JS Controllers, you can read more about them [here](https://docs.adonisjs.com/guides/controllers). You can also read more about the `moveToDisk` method [here](https://docs.adonisjs.com/guides/file-uploads#movetodisk).

3. It's better to keep track of the files you upload to Spaces. So we'll create a new Model called `File` with the following code:

```typescript
import Drive from '@ioc:Adonis/Core/Drive'
import {
  BaseModel,
  BelongsTo,
  belongsTo,
  column,
} from '@ioc:Adonis/Lucid/Orm'
import { ExpirySeconds, getExpiryPerType } from 'App/Utils/Files'
import { DateTime } from 'luxon'

import User from './User'

export default class File extends BaseModel {
  @column({ isPrimary: true, serializeAs: null })
  public id: number

  @column()
  public uid: string

  @column()
  public title: string

  @column({ serializeAs: null })
  public path: string

  @column()
  public type: string

  @column()
  public duration: number

  @column()
  public uri: string

  @column.dateTime({ autoCreate: true })
  public createdAt: DateTime

  @column.dateTime({ autoCreate: true, autoUpdate: true })
  public updatedAt: DateTime

  // Relationships
  @column({ serializeAs: null })
  public userId: number

  @belongsTo(() => User)
  public user: BelongsTo<typeof User>
}
```

Remember that you also need a migration file for your model, If you're not familiar with Adonis JS Models, you can read more about them [here](https://docs.adonisjs.com/guides/models/introduction).

4. Now we can update the `FilesController` to create a new `File` record after we upload the file to Spaces:

```typescript
import File from 'App/Models/File'

export default class FilesController {
  public async create({ auth, request }: HttpContextContract) {
    ...
    // Upload the file to Spaces
    await payload.file.moveToDisk(
      `./${dirPath}`,
      {
        visibility: Env.get('DRIVE_DISK') === 'local' ? 'public' : 'private',
      },
      Env.get('DRIVE_DISK'),
    )

    // Create a File record to keep track of the files we upload to Spaces
    const file = await File.create({
      uid: fileUID,
      title: payload.file.clientName,
      path: `${dirPath}/${payload.file.fileName}`,
      type: payload.type,
      duration: payload.file.duration,
      uri: Drive.getUrl(`${dirPath}/${payload.file.fileName}`),
      userId: user.id,
    })
    
    return {
      file
    }
  }
}
```

5. We can now create a new Route to upload files to Spaces in the `start/routes.ts` file:

```typescript
Route.group(() => {
  Route.post('/files', 'FilesController.create')
}).middleware('auth')
```

6. Now we can test our API using a cURL request to upload a file to Spaces:

```bash
curl -v --location --request POST 'http://localhost:3333/files' \
    --form 'file=@"/home/erfan/avatar.jpg"' \
    --form 'type="avatar"'
```

## Bonus: Signed URLs
Signed URLs are a great way to give temporary access to a file. You can read more about them on [official DigitalOcean Docs](https://docs.digitalocean.com/glossary/pre-signed-url/).

I personally like to store my signed URIs on a Redis instance and then only refresh them when expired. This way we can reduce the load it causes to generate these URIs when files are requested in bulk. For example when a user visits Netflix's homepage, they don't want to generate the private URIs for all the thumbnails at once, so these signed and secure URIs are cached and returned to the users upon their request.

1. Install and configure the Redis driver for Adonis JS

```bash
npm i @adonisjs/redis

node ace configure @adonisjs/redis
```

{{< quote author="Erfan Besharat" source="Wasted Hours on Debugging :D">}}
If you're using the DigitalOcean's hosted Redis instances and also hosting your app on their infrastructure, be sure to set a empty object for `tls` in `config/redis.ts`. You're using a private VPC anyway in that case.
```
connections: {
    local: {
      host: Env.get('REDIS_HOST'),
      port: Env.get('REDIS_PORT'),
      username: Env.get('REDIS_USER'),
      password: Env.get('REDIS_PASSWORD', ''),
      tls: {},
      db: 0,
      keyPrefix: '',
    },
  },
```
{{< /quote >}}

2. With all the Redis configs set and ready, you should now update your `File` model:
```typescript
import Redis from '@ioc:Adonis/Addons/Redis'
import Drive from '@ioc:Adonis/Core/Drive'
import {
  BaseModel,
  BelongsTo,
  HasMany,
  afterFetch,
  afterFind,
  afterPaginate,
  belongsTo,
  column,
  hasMany,
} from '@ioc:Adonis/Lucid/Orm'
import { ExpirySeconds, getExpiryPerType } from 'App/Utils/Files'
import { DateTime } from 'luxon'


export default class File extends BaseModel {
  ...
  @afterFetch()
  @afterFind()
  @afterPaginate()
  public static async loadThumbnailURI(files: File[]) {
    if (!Array.isArray(files)) {
      files = [files]
    }

    // Iterate if array otherwise only one file
    for (const file of files) {
      file.uri = file.path ? await fileURIFromRedis(file) : ''
    }
  }
  ...
}
```
If you're not familiar with [Adonis JS Hooks](https://docs.adonisjs.com/guides/models/hooks), be sure to check their docs first. In this part of code we added a `loadThumbnailURI` hook that runs on all `SELECT` queries and would fetch the signed URI for all the selected files using `fileURIFromRedis` function.

3. Now it's time to write our `fileURIFromRedis` function, add the following code to your `File` model as well, but make sure it's ourside the `File` class.

```typescript
const fileURIFromRedis = async (file: File): Promise<string> => {
  const pipeline = Redis.pipeline()
  const redisQuery = pipeline.get(`file:${file.uid}:uri`)
  const results = await redisQuery.exec()

  if (results && results.length > 0 && results[0][0]) {
    return results![0][1] as string
  }

  const expiry = getExpiryPerType(file.type)
  const uri = await Drive.getSignedUrl(file.path, {
    expiresIn: expiry,
  })

  pipeline.set(`file:${file.uid}:uri`, uri)
  pipeline.expire(`file:${file.uid}:uri`, ExpirySeconds[expiry])

  await pipeline.exec()

  return uri
}
```

5. Now everytime that you query your files, the URI field would be updated with the signed URI of the file from Redis.

## Conclusion
Using the above snippets you should be able to setup DigitalOcean spaces and Adonis JS to store your files on Cloud and also if you've completed the Bonus section, your files are also using secure and cached URIs to make your application more suitable for production environments.
