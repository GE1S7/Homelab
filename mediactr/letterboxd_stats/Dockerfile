FROM python:3.12.3-slim
RUN useradd --create-home --shell /bin/bash app_user
WORKDIR /home/app_user
RUN pip install --no-cache-dir letterboxd_stats
USER app_user
CMD ["/bin/bash", "-c", "bash;letterboxd_stats -d"]


