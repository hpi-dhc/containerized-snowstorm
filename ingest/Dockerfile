FROM python:3.10.6-slim
LABEL maintainer="Nictiz <info@snomed.nl>"

RUN pip install --no-cache-dir requests_toolbelt==0.9.1

# Create a working directory
RUN mkdir /releases \
    && mkdir /app
WORKDIR /app

COPY ingest.py /app/
ENV release=NULL
ENV rf2_files=NULL

CMD ["/bin/bash", "-c", "python ingest.py http://snowstorm:8080/ SNAPSHOT \"$rf2_files\" MAIN $release"]
#ENTRYPOINT ["python", "ingest.py"]