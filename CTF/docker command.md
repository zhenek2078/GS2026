# docker compose

Выполняем в директории, где лежит docker-compose.yml.

## build

### Просто собрать

```
docker compose build
```

### Собрать и запустить

```
docker compose up -d --build
```

## rebuild

### Просто пересобрать контейнер

```
docker compose up -d --build
```

### Пересобрать полностью, почистив кэш

```
docker compose build --no-cache
docker compose up -d
```

или

```
docker compose up -d --build --force-recreate
```

### Полный сброс

```
docker compose down
docker compose build --no-cache
docker compose up -d
```

# docker-compose

## build

### Просто собрать

```
docker-compose build
```

### Собрать и запустить

```
docker-compose up -d --build
```

## rebuild

### Просто пересобрать контейнер

```
docker-compose up -d --build
```

### Пересобрать полностью , почистив кэш

```
docker-compose build --no-cache
docker-compose up -d
```

### Полный сброс

```
docker-compose down
docker-compose up -d --build
```
