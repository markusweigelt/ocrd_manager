# OCR-D Manager

OCR-D Manager is a server that mediates between [Kitodo](https://github.com/kitodo) and [OCR-D](https://ocr-d.de). It resides on the site of the Kitodo installation (so the actual OCR server can be managed independently) but runs in its own container (so Kitodo can be managed independently).

Specifically, it gets called by [Kitodo.Production](https://github.com/kitodo/kitodo-production) or [Kitodo.Presentation](https://github.com/kitodo-presentation) to handle OCR for a document, and in turn calls the [OCR-D Controller](https://github.com/bertsky/ocrd_controller) for workflow processing. 

For an integration as a **service container**, orchestrated with other containers (Kitodo+Controller), see [this meta-repo](https://github.com/markusweigelt/kitodo_production_ocrd).

OCR-D Manager is responsible for
- data transfer from Kitodo to Controller and back,
- delegation to Controller,
- signalling/reporting,
- result validation,
- result extraction (putting ALTO files in the process directory where Kitodo expects them).

It is currently implemented as SSH login server with an installation of [OCR-D core](https://github.com/OCR-D/core) and an SSH client to connect to the Controller.

 * [Usage](#usage)
   * [Building](#building)
   * [Starting and mounting](#starting-and-mounting)
   * [General management](#general-management)
   * [Processing](#processing)
   * [Data transfer](#data-transfer)
   * [Logging](#logging)
   * [Monitoring](#monitoring)
 * [Testing](#testing)

## Usage

### Building

Build or pull the Docker image:

    make build # or docker pull markusweigelt/ocrd_manager

### Starting and mounting

Then run the container – providing a **host-side directory** for the volume …

 * `DATA`: directory for data processing (including images or existing workspaces),  
   defaults to current working directory

… but also files …

 * `KEYS`: public key **credentials** for log-in to the manager
 * `PRIVATE`: private key **credentials** for log-in to the controller …
 
… and (optionally) some **environment variables** …

 * `UID`: numerical user identifier to be used by programs in the container  
    (will affect the files modified/created); defaults to current user
 * `GID`: numerical group identifier to be used by programs in the container  
    (will affect the files modified/created); defaults to current group
 * `UMASK`: numerical user mask to be used by programs in the container  
    (will affect the files modified/created); defaults to 0002
 * `PORT`: numerical TCP port to expose the SSH server on the host side  
    defaults to 9022 (for non-priviledged access)
 * `CONTROLLER` network address:port for the controller client
			(must be reachable from the container network)
 * `ACTIVEMQ` network address:port of ActiveMQ server listening to result status
			(must be reachable from the container network)
 * `NETWORK` name of the Docker network to use  
    defaults to `bridge` (the default Docker network)

… thus, for **example**:

    make run DATA=/mnt/workspaces MODELS=~/.local/share KEYS=~/.ssh/id_rsa.pub PORT=9022 PRIVATE=~/.ssh/id_rsa

(You can also run the service via `docker-compose` manually – just `cp .env.example .env` and edit to your needs.)

### General management

Then you can **log in** as user `ocrd` from remote (but let's use `manager` in the following – 
without loss of generality):

    ssh -p 9022 ocrd@manager bash -i

(Typically though, you will run a non-interactive script, see next section.)

### Processing

In the manager, you can run shell scripts that do
- data management and validation via `ocrd` CLIs
- OCR processing by running workflows in the controller via `ssh ocrd@ocrd_controller` log-ins

The data management will depend on which Kitodo context you want to integrate into (Production 2 / 3 or Presentation).

For Kitodo.Production, there is a preconfigured script `for_production.sh` which takes the following arguments:
1. process ID
2. task ID
3. directory path
4. language
5. script
6. workflow name

The last argument (6) is optional and defaults to the preconfigured script `ocr.sh` which contains a trivial workflow:
- import of the images into a new OCR-D workspace
- preprocessing, layout analysis and text recognition with a single Tesseract processor call
- format conversion of the result from PAGE-XML to ALTO-XML

It can be replaced with the (path) name of any workflow script mounted under `/data`.

For example (assuming `testdata` is a directory with image files mounted under `/data`):

    ssh -T -p 9022 ocrd@manager for_production.sh 1 3 testdata deu Fraktur ocr.sh


### Logging

All logs are accumulated on standard output, which can be inspected via Docker:

    docker logs ocrd_controller

### Monitoring

The repo also provides a web server featuring
- (intermediate) results for all current document workspaces (via [OCR-D Browser](https://github.com/hnesk/browse-ocrd))
- :construction: log viewer
- :construction: task viewer
- :construction: workflow editor

Build or pull the Docker image:

    make build-monitor # or docker pull bertsky/ocrd_monitor

Then run the container – providing the same variables as above:

    make run-monitor DATA=/mnt/workspaces

You can then open `http://localhost:8080` in your browser.

## Testing

After [building](#building) and [starting](#starting-and-mounting), you can use the `test` target
for a round-trip:

    make test DATA=/mnt/workspaces

This will download sample data and run the default workflow on them.
