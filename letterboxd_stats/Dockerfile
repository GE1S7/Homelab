FROM python:3.8-slim
RUN useradd --create-home --shell /bin/bash app_user
WORKDIR /home/app_user
RUN pip install --no-cache-dir letterboxd_stats
USER app_user
CMD ["bash"]


