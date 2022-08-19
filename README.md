**This repository contains all the necessary scripts to set up and access SNOMED via *Snowstorm* and the *SNOMED CT Browser* on your own machine with just one single line of code.**

In the background, it creates and spins up the following four containerized applications:
1) A container with an *elasticsearch* database for storing and effectively accessing the ontology;
2) A container with an instance of the *Snowstorm* API to access the SNOMED ontology of the database in a standardized way;
3) A container with a script to load the ontology files into the database;
4) A container with the *SNOMED CT Browser* for graphical display.

**0) Prerequisites:**
- An up-to-date Docker instance (tested with `20.10.17)`.
- `git` installed.
- A stable internet connection.
- Your own copy of the SNOMED 'RF2' release files as *.zip* file.
- Only for Linux users: Please provide Docker and its corresponding user group(s) read AND write access to your file system (specifically: To the folders with the input data plus all its subdirectories, and to the folder you will be storing the *elasticsearch* database files in, cf. `PATH_TO_ELASTIC_DB_STORAGE_LOCATION` in this README and in the *example.env* file).

# Initial setup (import data)

##  **TL;DR - The short version**

Set the environment variables in `.env`; spin up the containers with
```
docker compose up --build
```
and access through port `80` (*Snowstorm*) or `8080` (*SNOMED CT Browser*).

## **The long version**

**1) Clone this repository:**

Use either of the following two commands:

```
git clone https://github.com/hpi-dhc/containerized-snowstorm.git
git clone git@github.com:hpi-dhc/containerized-snowstorm.git
```

**2) Set the environment variables:**

Before starting, folder paths, filenames, and SNOMED release details have to be specified in a separate *.env* file.
The accompanying *example.env* in this repository may serve as a template for it, however it should be renamed to *.env* thereafter.

The first two path variables refer to the paths used to store the input (RF2 *.zip* files) and database data.
They must be specified as absolute or relative paths on the host machine, with neither brackets nor quotations marks needed:
- `PATH_TO_ELASTIC_DB_STORAGE_LOCATION` : The path to the folder where the *elasticsearch* database files will be stored. This folder must first be created on the host machine before continuing with the next step of the setup. It will be a *read-and-write* volume, as the database needs to write its files to it during the ETL process. Subsequently, Docker has to be given *read-and-write* access to it (cf. prerequisites section above).
- `PATH_TO_RF2_FILES` : The path to the folder where the zipped (!) SNOMED release files are stored. This folder will be mounted just for the ETL of the database, not for production. As such, it will be *read-only* so that the SNOMED files will remain unchanged.

The following two variables should be adapted to the actual SNOMED version which is planned to be used; again with all variable names not enclosed by either brackets or quotations marks:
- `rf2_filename` : Should be the full name of the zipped RF2 files including the file extension.
- `release` : Should refer to the specific release of SNOMED (e.g., *SNOMED-INT* for the international version or *SNOMED-US* for the US-specific release).

From now on, make sure that the working directory is your repository path.

**3) Build and compose orchestrated containers:**

Use *docker compose* to run all four services:

a) The *`elasticsearch`* database service,
b) the *`snowstorm`* API service,
c) the *`ingest-rf2`* ETL service, and
d) the *`snomedct-browser`* service:
```
docker compose up --build
```

Building and downloading the images, spinning up the containers and executing the ETL scripts will take a considerable amount of time (50-70 minutes on a non-representative set of regular notebooks).

Wait for the process to finish.

*(A few comments):*
- *Closing the lid of a laptop or, more generally, sending the host machine to sleep might compromise the successful completion of this step.*
- *Not detaching (would be* `-d` *in the command above) from the terminal output will ensure traceability of the import status, but be prepared for a lot of repeated output from the `elasticsearch` service, especially regarding security features (cf. Notes section below).*
- *The terminal output of the `ingest-rf2` container periodically shows an* `exited with code 1` *at the beginning. This is intended, as this container needs to fail (to restart) before both the `elasticsearch` and `snowstorm` containers are up and running. A more elegant way would be to use a* `wait-for` *script in the `ingest-rf2` container, but the outcome would be the same.*

**4) Shut down the orchestrated containers:**

After finishing, shut down the orchestrated containers by executing the following command:

```
docker compose down
```

If you are using the same window/tab of your terminal, you first need to `Ctrl` + `c` to detach from the process and come back to your working directory OR you open a new window/tab with the same working directory and execute the above command there.
Either way will work.

In any case, the database files will be persisted (at the location specified earlier) and accessed whenever you will start the containers anew.

**5) Uncomment the *`ingest-rf2`* service from the *docker compose* file**

Before you start your containers again, make sure you uncomment the entire *`ingest-rf2`* service section (lines 3-12) from the *docker-compose.yml* (e.g., in VS Code `"Ctrl" + "/"` on Windows/Linux (`"Cmd" + "/"` on Mac respectively).

**6) Back up your database files**

Just to be on the safe side, back up the entire folder at `PATH_TO_ELASTIC_DB_STORAGE_LOCATION`, e.g., by simply using

```
cp -r PATH_TO_ELASTIC_DB_STORAGE_LOCATION PATH_TO_BACKUP
```
and replacing the capitalized variables with the respective absolute or relative paths on the host machine.

**7) OPTIONAL: Set Snowstorm to be a database read-only service**

The entire data loading process is triggered by the *`ingest-rf2`* service, but is executed through the *`snowstorm`* service against the *elasticsearch* database.
In order to prevent inadvertent changes to the database, comment out line 37 and uncomment line 38 of the *docker-compose.yml* file (which includes the `--snowstorm.rest-api.readonly=true` flag in the entrypoint).
This will only allow API endpoints with reading capabilities to be accessible.


# Regular operation (accessing data once imported)


**1) Start the orchestrated containers**

**IMPORTANT:** Make sure you have uncommented the entire *`ingest-rf2`* service section in the *docker-compose.yml* file as indicated in step 5 above.
Not following this procedure will likely result in exiting containers.

With this modified *docker-compose.yml*, execute

```
docker compose up -d
```

**2) Start querying with a) *Snowstorm* or b) *SNOMED CT Browser***

a) *Snowstorm*:
Open a browser on your local machine and type `localhost:8080` to access the Swagger UI interface of *Snowstorm* in the same way as you can access [the Swagger UI from SNOMED International](https://snowstorm.ihtsdotools.org/snowstorm/snomed-ct/swagger-ui.html).
The variable `branch` (if required) is `MAIN`.

b) *SNOMED CT Browser*:
Open a browser on your local machine and type `localhost:80` to access the *SNOMED CT Browser* graphical user interface to SNOMED, allowing you to access the ontology in the same way as through the [publicly available SNOMED CT Browser](https://browser.ihtsdotools.org/).
A minor issue still is the naming of the version in the UI, currently stating 'undefined'.
Nevertheless, all features are available.

Other variants of accessing the API exist, cf. Notes section below.

Happy querying! :)

**3) Shut down services after usage**

If you are done querying and want to spare resources on your computer, stop the containers with

```
docker compose down
```

# Notes:

- The memory estimates of 24 GB for the *`snowstorm`* service (lines 37 ff. in *docker-compose.yml*) will ensure an uninterrupted loading process, but are likely an overestimation that has not been systematically determined in experiments. On the other hand, the default values from the *Snowstorm* repository have turned out to be on the lower end (for a notebook). You should also experiment with the memory allocation in the Docker settings, which should be larger than the combined allocated memory for all three/four services.
- Instead of using the *Swagger UI* to access the *Snowstorm* API, you can access the API automatically via any other script, e.g., [this excellent set of functions](https://github.com/mertenssander/python_snowstorm_client) implemented in Python.
    - Any option accessing *Snowstorm* will need to go through port `8080`:
        - Through `localhost:8080` if you are accessing the API from your local machine (host port number can be changed in the ports section of the *`snowstorm`* service in the *docker-compose.yml* file, e.g., when the port is already taken on your host machine);
        - Through `snowstorm:8080` after attaching to the Docker network `elastic` (network name can be changed in line 57 of the same file) or after directly including the new service into the *docker-compose.yml* file (however in these cases changing ports of the *`snowstorm`* service to be exposed inside the Docker network is not straightforward, as the port settings would also need to be changed within the container).
- If not otherwise specified, all commands are executed on the host machine with the working directory being the cloned repository.
- This repository is intended for local use only. Deployment in a production setting would require additional security mechanisms.
- One of these mechanisms would be to pin versions (e.g., of the *`snomedct-browser`* service if provided by the publisher) to ensure exact reproducibility.
- The files in the `ingest` folder are based on the folder with the same name in the Nictiz repository (https://github.com/Nictiz/Nictiz-Snowstorm), but have undergone a few modifications (traced in the *ingest.py*).
- The `docker-compose.yml` file is a heavy modification of the identically named file from the IHTSDO GitHub repository at https://github.com/IHTSDO/snowstorm
- This project gratefully builds upon the existing work from [Nictiz](https://www.nictiz.nl/) and [IHTSDO/SNOMED International](https://www.snomed.org/), who kindly provide their code under an Apache 2.0 license. So does this project. Nevertheless, all users are kindly invited to contribute to the project, specifically to leave a note to the author if you find parts of the code to be broken or the explanations in this README ambiguous.

### Acknowledgements

Thanks to [Thomas Harris](https://github.com/0teh) for investigating and suggesting how to modify the *`ingest-rf2`* service such that all database content is also visible to the *`snomedct-browser`* service.


*(Written by [Jan Philipp Sachs](www.jpsachs.de); updated on August 19, 2022)*
