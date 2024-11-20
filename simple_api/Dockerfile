FROM python:3.8-buster
LABEL maintainer="zakarieh"
COPY student_age.py /
RUN DEBIAN_FRONTEND=noninteractive apt update -y && apt install python-dev python3-dev libsasl2-dev python-dev libldap2-dev libssl-dev -y
COPY requirements.txt /requirements.txt
RUN pip3 install -r /requirements.txt
RUN mkdir /data
VOLUME ["/data"]
EXPOSE 5000
CMD ["python3", "./student_age.py"]
