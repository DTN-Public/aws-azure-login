[![view on npm](http://img.shields.io/npm/v/aws-azure-login.svg)](https://www.npmjs.org/package/aws-azure-login)
[![npm module downloads per month](http://img.shields.io/npm/dm/aws-azure-login.svg)](https://www.npmjs.org/package/aws-azure-login)

# aws-azure-login
If your organization uses [Azure Active Directory](https://azure.microsoft.com) to provide SSO login to the AWS console, then there is no easy way to log in on the command line or to use the [AWS CLI](https://aws.amazon.com/cli/). This tool fixes that. It lets you use the normal Azure AD login (including MFA) from a command line to create a federated AWS session and places the temporary credentials in the proper place for the AWS CLI and SDKs.

## Installation

### Windows
Install [Node.js](https://nodejs.org/) v7.6.0 or higher. Then install aws-azure-login with npm:

    npm install -g aws-azure-login

### Linux

In Linux you can either install for all users or just the current user. In either case, you must first install [Node.js](https://nodejs.org/) v7.6.0 or higher and any [puppeteer dependencies](https://github.com/GoogleChrome/puppeteer/blob/master/docs/troubleshooting.md#chrome-headless-doesnt-launch). Then follow the appropriate instructions. 

#### Option A: Install for All Users

Install aws-azure-login globally with npm:

    sudo npm install -g aws-azure-login --unsafe-perm
    
Puppeteer doesn't install globally with execution permissions for all users so you'll need to modify them:

    sudo chmod -R go+rx $(npm root -g)

#### Option B: Install Only for Current User

First configure npm to install global packages in [your home directory](https://docs.npmjs.com/getting-started/fixing-npm-permissions):
   
    mkdir ~/.npm-global
    npm config set prefix '~/.npm-global'
    export PATH=~/.npm-global/bin:$PATH
    source ~/.profile
    echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.profile
    source ~/.profile
     
Then install aws-azure-login:

    npm install -g aws-azure-login

### Docker

A Docker image has been built with aws-azure-login preinstalled. You simply need to run the command with a volume mounted to your AWS configuration directory.

    docker run --rm -it -v ~/.aws:/root/.aws sportradar/aws-azure-login
    
The Docker image is configured with an entrypoint so you can just feed any arguments in at the end.

You can also put the docker-launch.sh script into your bin directory for the aws-azure-login command to function as usual:

    sudo curl -o /usr/local/bin/aws-azure-login https://raw.githubusercontent.com/sportradar/aws-azure-login/master/docker-launch.sh
    sudo chmod o+x /usr/local/bin/aws-azure-login

Now just run `aws-azure-login`.

## Usage

### Configuration

#### AWS

To configure the aws-azure-login client run:

    aws-azure-login --configure
    
You'll need your Azure Tenant ID and the App ID URI. To configure a named profile, use the --profile flag.

    aws-azure-login --configure --profile foo

#### Environment Variables

You can optionally store your responses as environment variables:

* `AZURE_TENANT_ID`
* `AZURE_APP_ID_URI`
* `AZURE_DEFAULT_USERNAME`
* `AZURE_DEFAULT_PASSWORD`
* `AZURE_DEFAULT_ROLE_ARN`
* `AZURE_DEFAULT_DURATION_HOURS`

To avoid having to ``<Enter>`` through the prompts after setting these environment variables, use the `--no-prompt` option when running the command.

    aws-azure-login --no-prompt

Use the `HISTCONTROL` environment variable to avoid storing the password in your bash history (notice the space at the beginning):

    $ HISTCONTROL=ignoreboth
    $  export AZURE_DEFAULT_PASSWORD=mypassword
    $ aws-azure-login

### Logging In

Once aws-azure-login is configured, you can log in. For the default profile, just run:

    aws-azure-login
    
You will be prompted for your username and password. If MFA is required you'll also be prompted for a verification code or mobile device approval. To log in with a named profile:

    aws-azure-login --profile foo

Alternatively, you can set the `AWS_PROFILE` environmental variable to the name of the profile just like the AWS CLI.

Once you log in you can use the AWS CLI or SDKs as usual!

If you are logging in on an operating system with a GUI, you can log in using the actual Azure web form instead of the CLI:

    aws-azure-login --mode gui

Logging in with GUI mode is likely to be much more reliable.

_Note:_ on Linux you will likely need to disable the Puppeteer sandbox or Chrome will fail to launch:

    aws-azure-login --no-sandbox

### Behind corporate proxy

If behind corporate proxy, then just set https\_proxy env variable.

## Getting Your Tenant ID and App ID URI

Your Azure AD system admin should be able to provide you with your Tenant ID and App ID URI. If you can't get it from them, you can scrape it from a login page from the myapps.microsoft.com page.

1. Load the myapps.microsoft.com page.
2. Click the chicklet for the login you want.
3. In the window the pops open quickly copy the login.microsoftonline.com URL. (If you miss it just try again. You can also open the developer console with nagivation preservation to capture the URL.)
4. The GUID right after login.microsoftonline.com/ is the tenant ID.
5. Copy the SAMLRequest URL param.
6. Paste it into a URL decoder ([like this one](https://www.samltool.com/url.php)) and decode.
7. Paste the decoded output into the a SAML deflated and encoded XML decoder ([like this one](https://www.samltool.com/decode.php)).
8. In the decoded XML output the value of the Issuer tag is the App ID URI.

## How It Works

The Azure login page uses JavaScript, which requires a real web browser. To automate this from a command line, aws-azure-login uses [Puppeteer](https://github.com/GoogleChrome/puppeteer), which automates a real Chromium browser. It loads the Azure login page behind the scenes, populates your username and password (and MFA token), parses the SAML assertion, uses the [AWS STS AssumeRoleWithSAML API](http://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithSAML.html) to get temporary credentials, and saves these in the CLI credentials file.

## Troubleshooting

The nature of browser automation with Puppeteer means the solution is bit brittle. A minor change on the Microsoft side could break the tool. If something isn't working, you can fall back to GUI mode (above). To debug an issue, you can run in debug mode (--mode debug) to see the GUI while aws-azure-login tries to populate it. You can also have the tool print out more detail on what it is doing to try to do in order to diagnose. aws-azure-login uses the [Node debug module](https://www.npmjs.com/package/debug) to print out debug info. Just set the DEBUG environmental variable to 'aws-azure-login'. On Linux/OS X:

    DEBUG=aws-azure-login aws-azure-login

On Windows:

    set DEBUG=aws-azure-login
    aws-azure-login

## Support for Other Authentication Providers

Obviously, this tool only supports Azure AD as an identity provider. However, there is a lot of similarity with how other logins with other providers would work (especially if they are SAML providers). If you are interested in building support for a different provider let me know. It would be great to build a more generic AWS CLI login tool with plugins for the various providers.
