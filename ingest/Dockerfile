FROM python:3.10.6-slim

LABEL maintainer="Jan Philipp Sachs" \
      email="jan-philipp.sachs@hpi.de" \
      institution="Hasso Plattner Institute, University of Potsdam, Germany"

RUN pip install --no-cache-dir requests_toolbelt==0.9.1

# Create a working directory
RUN mkdir /releases \
    && mkdir /app
WORKDIR /app

COPY ingest.py /app/
ENV release=NULL
ENV rf2_filename=NULL

CMD ["/bin/bash", "-c", "python ingest.py http://snowstorm:8080 SNAPSHOT \"$rf2_filename\" MAIN $release"]
#ENTRYPOINT ["python", "ingest.py"]