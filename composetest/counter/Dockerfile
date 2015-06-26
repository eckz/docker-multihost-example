FROM python:2.7
RUN pip install flask redis
COPY . /code
WORKDIR /code
CMD ["python", "app.py"]
