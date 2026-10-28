# randomquotes - Tutorial 06

Microservicio Express que devuelve una frase aleatoria, desplegado con Docker en una instancia AWS EC2.

## Despliegue
```bash
docker build -t randomquotes-juan-pena:1.0 .
docker run -d --name randomquotes-juan-pena --restart unless-stopped -p 8083:80 randomquotes-juan-pena:1.0
```

## Evidencia
La aplicación corriendo en AWS (http://IP:8083). Cada recarga muestra una frase distinta:

