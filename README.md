# Terragate
[![Docker](https://github.com/Jandini/Terragate/actions/workflows/docker-image.yml/badge.svg)](https://github.com/Jandini/Terragate/actions/workflows/docker-image.yml)

Simple gateway API for terraform command line tool.










# Development 

This is a summary of all challenges and lessons learned while developing this project.



## Docker port mapping must come before image name

The port mapping option `-p` MUST be provided before image name. 

* Correct format. Port option has to come before image name.
  ```sh
  docker run -p 3000:80 my_image
  ```

* Wrong format. Docker will not notify you, and will silently fail port publishing.

  ```sh
  docker run my_image -p 3000:80
  ```




## How to execute command inside container

Start sample container in `-d` detached mode with `--rm` flat to remove container once is stopped.

```sh
docker run -d --rm -p 8000:80 --name aspnetcore_sample mcr.microsoft.com/dotnet/samples:aspnetapp
```

The new container id starts with `ba`.

```
ba04d41df2187e1d2efaf367f48be32934cc2b716f4e972bfd066e1106a084f9
```

Execute `ls` command from `ba`... container with `-it` interactive mode.

```sh
docker exec -it ba ls
```

Current folder content is displayed.

```sh
appsettings.Development.json  aspnetapp.pdb                                                       
appsettings.json              wwwroot
aspnetapp
```



## Download terraform inside container

By default base images like `mcr.microsoft.com/dotnet/aspnet:7.0 ` do not have tools like `wget` or `unzip`. You must install them in order to use them to download other prerequisites. 

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:7.0 
WORKDIR /app

# Install prerequisites
RUN apt-get update && \
    apt-get install -y wget unzip 

# Download and unpack terraform
RUN wget https://releases.hashicorp.com/terraform/1.3.7/terraform_1.3.7_linux_amd64.zip && \
    unzip terraform_1.3.7_linux_amd64.zip && \
    chmod +x terraform

ENTRYPOINT ./terraform
```

**The certificate of 'releases.hashicorp.com' is not trusted. ** may be thrown when certificates are not updated. Make sure there certificates are provided and certificate repository is updated. 

```
 > [4/8] RUN wget https://releases.hashicorp.com/terraform/1.3.7/terraform_1.3.7_linux_amd64.zip:
 #7 0.364 --2023-01-22 15:58:36--  https://releases.hashicorp.com/terraform/1.3.7/terraform_1.3.7_linux_amd64.zip
 #7 0.372 Resolving releases.hashicorp.com (releases.hashicorp.com)... 108.138.85.30, 108.138.85.31, 108.138.85.65, ... 
 #7 0.660 Connecting to releases.hashicorp.com (releases.hashicorp.com)|108.138.85.30|:443... connected. 
 #7 1.187 ERROR: The certificate of 'releases.hashicorp.com' is not trusted. 
 #7 1.187 ERROR: The certificate of 'releases.hashicorp.com' doesn't have a known issuer. 
```

Ensure the certificates are provided and updated before download command is executed.

```dockerfile
COPY *.crt /usr/local/share/ca-certificates/
RUN update-ca-certificates
RUN wget https://releases.hashicorp.com/terraform/1.3.7/terraform_1.3.7_linux_amd64.zip 
```



## Terraform will not work without certificates

**Failed to query available provider packages** error is throw when terraform cannot reach out to the registry. 

```sh
│ Error: Failed to query available provider packages
│
│ Could not retrieve the list of available versions for provider vmware/vra7: could not connect to registry.terraform.io: Failed to request discovery document: Get
│ "https://registry.terraform.io/.well-known/terraform.json": x509: certificate signed by unknown authority
```

This indicate that your organization may be using something like Zscaler.  Add required certificate to `Dockerfile`.

``` dockerfile
COPY YOUR_CERTIFICATE.crt /usr/local/share/ca-certificates/YOUR_CERTIFICATE.crt
RUN update-ca-certificates
```

**Unable to get auth token** error indicate that you are still missing your organization certificates. 


```sh
│ Error: Error: Unable to get auth token: Post "https://cloud.xyz.com/identity/api/tokens": x509: certificate signed by unknown authority  
│
│   with provider["registry.terraform.io/vmware/vra7"],
│   on test.tf line 10, in provider "vra7":
│   10: provider "vra7" {
```

Terraform is not able to connect to the host specified in the provider. 

```json
provider "vra7" {
  username = ""
  password = ""
  tenant   = ""
  host     = "https://cloud.xyz.com"
}
```

Having to add multiple certificates can be resolved this way:

```dockerfile
COPY *.crt /usr/local/share/ca-certificates/
RUN update-ca-certificates
```

The copy will find all the files (recursively) from root of your repository and copy them to desired output. 

**IMPORTANT:**  The certificates must have CRT extension in `/usr/local/share/ca-certificates/`. Otherwise update-ca-certificates will not find them.



## Reduce docker image size  

Removing files `rm terraform_1.3.7_linux_amd64.zip` reduces image size. Each RUN creates a docker image layer. The break line character `&& \` allows to execute multiple commands and create one docker image layer.  This is example of terraform running in alpine Linux. 

```dockerfile
FROM alpine:3.14
WORKDIR /app

# Install prerequisites
RUN apk add --no-cache wget unzip

# Download and unpack terraform
RUN wget https://releases.hashicorp.com/terraform/1.3.7/terraform_1.3.7_linux_amd64.zip && \
    unzip terraform_1.3.7_linux_amd64.zip -d /usr/bin && \
    rm terraform_1.3.7_linux_amd64.zip && \
    chmod +x /usr/bin/terraform

ENTRYPOINT terraform
```

This container has only 69.4MB because of `alpine:3.14` image and zip file was deleted. 


```
REPOSITORY           TAG                      IMAGE ID       CREATED          SIZE                                                     jandinis/terragate   0.1.0-terraform-file.1   f2507eb25f58   7 seconds ago    69.4MB   
```



## How to export certificates from local machine

If terraform works on your machine, you can find and export required certificates. 

1. Open Manage user certificates or run `certlm.msc`.
2. Expand `Trusted Root Certification Authrorities` tree node.
3. Find your certificate and from context menu select `All Tasks > Export...`
4. Click `Next` in Welcome to the Certificate Export Wizard
5. Select `Base-64 encoded X.509 (.CER)` format. 
6. Save the file. Be aware that the extension will be changed to *.cer. The docker image is expecting *.crt. 






## Docker build fails "COPY failed: no source files were specified"

When running docker COPY and files are not present it will fail. 

```
COPY res/*.crt "/usr/local/share/ca-certificates/"
```

will result with 

```
Step 5/11 : COPY res/*.crt "/usr/local/share/ca-certificates/"
COPY failed: no source files were specified
Error: Process completed with exit code 1.
```

Using `/target` parameter does not help here. 

Based on https://stackoverflow.com/a/54245466 I created conditional COPY in docker file based on given `ADD_CA_CERTS` argument: 

```dockerfile
# Set this argument to 1 or 0 to include or not include ca certificates from res 
ARG ADD_CA_CERTS=0
...

# Copy certificates only if ADD_CA_CERTS argument is set to 1
FROM base as ca-certs-1
ONBUILD COPY res/*.crt /usr/local/share/ca-certificates/
ONBUILD RUN update-ca-certificates

# if ADD_CA_CERTS is empty or is set to 0 then only echo the message.
FROM base as ca-certs-
FROM base as ca-certs-0
ONBUILD RUN echo "Build without ca certificates"

# Trigger COPY and RUN defined in "ca-cert-1" when ADD_CA_CERTS variable is set to 1.
FROM ca-certs-${ADD_CA_CERTS}

...
```

After this I learnt about [Docker COPY vs Docker ADD? | Jhooq](https://jhooq.com/docker-copy-vs-docker-add/)



## GitVersion fails in GitHub Actions

Running `dotnet gitversion` fails with `System.NullReferenceException: Object reference not set to an instance of an object.` when running it in GitHub Actions.

```yaml
- name: Run GitVersion
  run: |  
    dotnet tool install --global GitVersion.Tool
    dotnet gitversion /output buildserver 
```

Use `fetch-depth: 0` in `actions/checkout@v3`

```yaml
- uses: actions/checkout@v3
  with: 
    fetch-depth: 0
```





## Publish fails with `project.assets.json` doesn't have a target for 'net7.0/linux-x64'. 



Running `dotnet restore` and `dotnet publish` separately: 

```yaml
 run: |  
   dotnet restore
   dotnet publish --no-restore -nologo --configuration Release --runtime linux-x64 --no-self-contained
```

throws following error:

```
... project.assets.json` doesn't have a target for 'net7.0/linux-x64'. Ensure that restore has run and that you have included 'net7.0' in the TargetFrameworks for your project. You may also need to include 'linux-x64' in your project's RuntimeIdentifiers.
```



No solution yet, but quick workaround is to run publish in one statement.

```
dotnet publish -nologo --configuration Release --runtime linux-x64 --no-self-contained
```

