FROM python:3.10-bullseye
RUN pip install --upgrade pip

RUN adduser --disabled-password worker
USER worker
WORKDIR /home/worker

COPY --chown=worker:worker ./requirements.txt requirements.txt
RUN pip install --user -r requirements.txt

ENV PATH="/home/worker/.local/bin:${PATH}"

COPY --chown=worker:worker . .