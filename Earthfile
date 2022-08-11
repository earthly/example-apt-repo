VERSION 0.6
FROM --platform=linux/amd64 ubuntu:20.04

hello-world-binary:
    WORKDIR /code
    RUN apt-get update && apt-get install -y gcc
    COPY hello.c .
    RUN gcc -o hello-world hello.c
    SAVE ARTIFACT hello-world
    
hello-world-package:
    WORKDIR /package/hello-world_0.0.1-1_amd64
    COPY control DEBIAN/.
    COPY +hello-world-binary/hello-world usr/bin/.
    RUN dpkg --build /package/hello-world_0.0.1-1_amd64
    SAVE ARTIFACT /package/hello-world_0.0.1-1_amd64.deb

generate-pgp-key:
    WORKDIR /pgp-key
    RUN apt-get update && apt-get install -y gpg
    RUN echo "%echo Generating an example PGP key
Key-Type: RSA
Key-Length: 4096
Name-Real: example
Name-Email: example@example.com
Expire-Date: 0
%no-ask-passphrase
%no-protection
%commit" > example-pgp-key.batch
    RUN gpg --no-tty --batch --gen-key example-pgp-key.batch
    RUN gpg --armor --export example > pgp-key.public
    RUN gpg --armor --export-secret-keys example > pgp-key.private
    SAVE ARTIFACT pgp-key.public
    SAVE ARTIFACT pgp-key.private

create-repo:
    WORKDIR /apt-repo
    RUN apt-get update && apt-get install -y dpkg-dev
    COPY generate-release.sh /root/bin/.
    RUN mkdir -p ./pool/main/
    RUN mkdir -p ./dists/stable/main/binary-amd64
    COPY +hello-world-package/hello-world_0.0.1-1_amd64.deb ./pool/main/binary-amd64/.

    # generate Packages and Packages.gz
    RUN dpkg-scanpackages --arch amd64 pool/ > dists/stable/main/binary-amd64/Packages
    RUN cat dists/stable/main/binary-amd64/Packages | gzip -9 > dists/stable/main/binary-amd64/Packages.gz

    # generate Release
    WORKDIR /apt-repo/dists/stable
    RUN /root/bin/generate-release.sh > Release

    # sign Release
    COPY +generate-pgp-key/pgp-key.private /.
    RUN cat /pgp-key.private | gpg --import
    RUN cat /apt-repo/dists/stable/Release | gpg --default-key example -abs > /apt-repo/dists/stable/Release.gpg
    RUN cat /apt-repo/dists/stable/Release | gpg --default-key example -abs --clearsign > /apt-repo/dists/stable/InRelease

    SAVE ARTIFACT /apt-repo AS LOCAL example-apt-repo

repo-server:
    RUN apt-get update && apt-get install -y python3
    WORKDIR /www
    COPY +create-repo/apt-repo /www/apt-repo
    CMD ["python3", "-m", "http.server"]


test:
    COPY +generate-pgp-key/pgp-key.public /example.pgp
    COPY docker-compose.yml .
    WITH DOCKER --compose docker-compose.yml --load=repo-server:latest=+repo-server
        RUN \
            echo "deb [arch=amd64 signed-by=/example.pgp] http://127.0.0.1:8000/apt-repo stable main" > /etc/apt/sources.list.d/example.list && \
            apt-get update && \
            apt-get install -y hello-world && \
            hello-world
    END
