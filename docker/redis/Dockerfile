# Usar a versão oficial do Redis
FROM redis:7.2

# Copiar o arquivo de configuração personalizado
COPY redis.conf /usr/local/etc/redis/redis.conf

# Definir o comando de inicialização com configuração segura
CMD ["redis-server", "/usr/local/etc/redis/redis.conf"]