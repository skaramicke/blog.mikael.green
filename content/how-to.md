---
title: How to use this project
menus: main
description: Getting started instructions for Hugo Blog on Cloudflare with Decap CMS
---

## Getting Started

### 1. Copy this repository

### 2. Configure Decap CMS

1. Edit the file `static/admin/config.yml`
   - Set the `repo` to the name of the repository you just created.
   - Set the `baseURL` to the https url to the domain you'll set up later, for instance "https://best-blog-in-the-world.com"

### 3. Create Github OAuth App

1. Create an app with an appropriate name for the blog in in [Github Developer Settings](https://github.com/settings/developers)

2. Set Callback to the https url to the domain you'll set up later, followed by /callback, for instance "https://best-blog-in-the-world.com/callback"

3. Grab the Client ID and create a Client Secret for the next step.

### 4. Configure Cloudflare Pages

_valid as of 2023-12-18_

1. Log in to CloudFlare, hit Workers & Pages in the left sidebar, and then Create application.

2. Click the Pages tab and then Connect to Git.

3. Connect GitHub if you haven't already, and choose the repository you just created.

4. Choose Hugo under Framwork Preset, and open the Environment Variable section.

5. Create the following environment variables:

- `OAUTH_CLIENT_ID` - The Client ID from the Github OAuth App
- `OAUTH_CLIENT_SECRET` - The Client Secret from the Github OAuth App
- `REDIRECT_URI` - Set this to the https url to the domain you'll set up later, followed by /callback, for instance "https://best-blog-in-the-world.com/callback"

6. Click Save and Deploy.

7. Set up your Custom Domain
