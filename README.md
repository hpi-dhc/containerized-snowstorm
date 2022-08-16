This repository provides the necessary scripts for setting up services to access the SNOMED ontology:
1) A container with an *elasticsearch* database for storing and effectively accessing the ontology;
2) A container with an instance of the *Snowstorm* API to access the SNOMED ontology of the database in a standardized way;
3) A container with a script to load the ontology files into the database.
<!-- 4) A container with the SNOMED CT Browser for graphical display. -->

**0) Prerequisites:**
- An up-to-date Docker instance (tested with `20.10.17)`.
- `git` installed.
- A stable internet connection.
- Your own copy of the SNOMED 'RF2' release files as *.zip* file.
- Only for Linux users: Please provide Docker and its corresponding user group(s) read AND write access to your file system (specifically: To the folders with the input data plus all its subdirectories, and to the folder you will be storing the elasticsearch database files in, cf. `PATH_TO_ELASTIC_DB_STORAGE_LOCATION` in this README and in the *example.env* file).

# Initial setup (import data)

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

Use *docker compose* to run all three services:

a) The *elasticsearch* database service,
b) the *Snowstorm* API service, and
c) the *ingest-rf2* script service:
<!-- Use *docker compose* to run all three services: a) The *elasticsearch* database service, b) the *Snowstorm* API service, c) the import script service, and d) the *SNOMED CT Browser* service: -->

```
docker compose up --build
```

Building and downloading the images, spinning up the containers and executing the ETL scripts will take a considerable amount of time (1-2 hours on a non-representative set of regular notebooks).

Wait for the process to finish.

*(A few comments):*
- *Closing the lid of a laptop or, more generally, sending the host machine to sleep might compromise the successful completion of this step.*
- *Not detaching (would be* `-d` *in the command above) from the terminal output will ensure traceability of the import status, but be prepared for a lot of repeated output from the elasticsearch service, especially regarding security features (cf. Notes section below).*

**4) Shut down the orchestrated containers:**

After finishing, shut down the orchestrated containers by executing the following command:

```
docker compose down
```

If you are using the same window/tab of your terminal, you first need to `Ctrl` + `c` to detach from the process and come back to your working directory OR you open a new window/tab with the same working directory and execute the above command there.
Either way will work.

In any case, the database files will be persisted (at the location specified earlier) and accessed whenever you will start the containers anew.

**5) Uncomment data ingest service from *docker compose* file**

Before you start your containers again, make sure you uncomment the entire `ingest-rf2` service section (lines 3-14) from the *docker-compose.yml* (e.g., in VS Code `"Ctrl" + "/"` on Windows/Linux (`"Cmd" + "/"` on Mac respectively).

**6) Back up your database files**

Just to be on the safe side, back up the entire folder at `PATH_TO_ELASTIC_DB_STORAGE_LOCATION`, e.g., by simply using

```
cp -r PATH_TO_ELASTIC_DB_STORAGE_LOCATION PATH_TO_BACKUP
```
and replacing the capitalized variables with the respective absolute or relative paths on the host machine.

**7) OPTIONAL: Set Snowstorm to be a database read-only service**

The entire data loading process is triggered by the *ingest-rf2* service, but is executed through the *Snowstorm* service against the *elasticsearch* database.
In order to prevent inadvertent changes to the database, comment out line 45 and uncomment line 46 of the *docker-compose.yml* file.
This will only allow API endpoints with reading capabilities to be accessible.


# Regular operation (accessing data once imported)


**1) Start the orchestrated containers**

**IMPORTANT:** Make sure you have uncommented the entire `ingest-rf2` service section in the *docker-compose.yml* file as indicated in step 5 above.
Not following this procedure will likely result in exiting containers.

With this modified *docker-compose.yml*, execute

```
docker compose up -d
```

**2) Start querying with *Snowstorm***

Open a browser on your local machine and type `localhost:8080` to access the Swagger UI interface of *Snowstorm*.
The variable `branch` (if required) is `MAIN`.

Other variants of accessing the API exist, cf. Notes section below.

Happy querying! :)

**3) Shut down services after usage**

If you are done querying and want to spare resources on your computer, stop the containers with

```
docker compose down
```

## Notes:

- The memory estimates of 24 GB for the Snowstorm service (lines 25-27 in *docker-compose.yml*) will ensure an uninterrupted loading process, but are likely an overestimation that has not been systematically determined in experiments. On the other hand, the default values from the Snowstorm repository have turned out to be on the lower end (for a notebook). You should also experiment with the memory allocation in the Docker settings, which should be larger than the combined allocated memory for all three services.
- Instead of using the Swagger UI to access the Snowstorm API, you have at least two other options:
    - **Option 1:** Access the API automatically via any other script, e.g. [this excellent set of functions](https://github.com/mertenssander/python_snowstorm_client) implemented in Python.
    - **Option 2:** You spin up a *SNOMED CT Browser* instance, allowing you to access the ontology in the same way as through the [IHTSDO SNOMED CT Browser web service](https://browser.ihtsdotools.org/).
    - Both options will either go through port `8080` (can be changed in the ports section of the *snowstorm* service in the *docker-compose.yml* file, e.g., when the port is already taken on your host machine) if you are accessing the API from your local machine, or through port `8080` after attaching to the Docker network `elastic` or after directly including the new service into the *docker-compose.yml* file (however in this case changing ports is not straightforward, as the port settings would also need to be changed within the corresponding container).
- If not otherwise specified, all commands are executed on the host machine with the working directory being the cloned repository.
- This repository is intended for local use only. Deployment in a production setting would require additional security mechanisms.
<!-- - One of these mechanisms would be to pin versions (e.g., of the *snomedct-browser*) to ensure exact reproducibility. -->
- The files in the `ingest` folder are based on the folder with the same name in the Nictiz repository (https://github.com/Nictiz/Nictiz-Snowstorm). While the `ingest.py` file has remained unchanged, the `Dockerfile` has undergone a few modifications.
- The `docker-compose.yml` file is a heavy modification of the identically named file from the IHTSDO GitHub repository at https://github.com/IHTSDO/snowstorm
- This project gratefully builds upon the existing work from [Nictiz](https://www.nictiz.nl/) and [IHTSDO/SNOMED International](https://www.snomed.org/), who kindly provide their code under an Apache 2.0 license. So does this project. Nevertheless, all users are kindly invited to contribute to the project, specifically to leave a note to the author if you find parts of the code to be broken or the explanations in this README ambiguous.


*(Written by [Jan Philipp Sachs](www.jpsachs.de) on August 15, 2022)*
