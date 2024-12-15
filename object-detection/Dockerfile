FROM anibali/pytorch:cuda-10.0

WORKDIR /ssd-det

COPY requirements.txt /ssd-det
RUN sudo apt-get update
RUN sudo apt-get install -y libsm6 libxext6 libxrender-dev libglib2.0-0
RUN pip install --upgrade pip setuptools wheel
RUN pip install -r ./requirements.txt

COPY . /ssd-det
RUN sudo chmod -R a+rwx /ssd-det/
CMD ["python", "-u", "app.py"]