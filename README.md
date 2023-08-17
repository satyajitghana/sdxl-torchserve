# Stable Diffusion XL - TorchServe

```
pip install -r requirements.txt
```

```
python3 download_model.py
bash zip_model.sh
```

```
docker pull pytorch/torchserve:0.8.1-gpu
```

```
docker run -it --rm --gpus all -v `pwd`:/opt/src pytorch/torchserve:0.8.1-gpu bash
```


```
torch-model-archiver --model-name sdxl --version 1.0 --handler sdxl_handler.py --extra-files sdxl-1.0-model.zip -r requirements.txt
```

```
docker run --rm --shm-size=1g \
        --ulimit memlock=-1 \
        --ulimit stack=67108864 \
        -p8080:8080 \
        -p8081:8081 \
        -p8082:8082 \
        -p7070:7070 \
        -p7071:7071 \
		--gpus all \
		-v /home/ubuntu/sdxl-torchserve/config.properties:/home/model-server/config.properties \
        --mount type=bind,source=/home/ubuntu/sdxl-torchserve,target=/tmp/models pytorch/torchserve:0.8.1-gpu torchserve --model-store=/tmp/models
```