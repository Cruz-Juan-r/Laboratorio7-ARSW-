# Laboratorio7-ARSW-

## Alta Disponibilidad con Application Load Balancer en AWS

| Campo | Valor |
|---|---|
| **Asignatura** | Arquitecturas de Software |
| **Duración sugerida** | 2 a 3 horas |
| **Modalidad** | Individual o parejas |
| **Plataforma** | AWS Academy Learner Lab |
| **Créditos disponibles** | 50 USD |
| **Restricción** | Sin creación ni uso de perfiles IAM personalizados |

## 1. Propósito

En los sistemas modernos no basta con desplegar una aplicación en un único servidor: si ese servidor falla, el sistema completo queda fuera de servicio. La **alta disponibilidad** busca reducir ese riesgo mediante redundancia, balanceo de carga, detección de fallos y distribución de componentes en varias zonas de disponibilidad.

Este laboratorio implementa una arquitectura básica de alta disponibilidad en AWS utilizando:

- Amazon EC2
- Application Load Balancer (ALB)
- Target Group
- Security Groups
- Health Checks
- Dos zonas de disponibilidad

## 2. Resultados de aprendizaje

Al finalizar, el estudiante estará en capacidad de:

- Explicar qué es alta disponibilidad.
- Diferenciar disponibilidad, tolerancia a fallos y escalabilidad.
- Crear una arquitectura redundante con dos instancias EC2.
- Configurar un Application Load Balancer.
- Crear un Target Group con health checks.
- Validar el balanceo de carga entre instancias.
- Simular una falla y observar el comportamiento del balanceador.
- Relacionar la solución con atributos de calidad como disponibilidad, resiliencia y mantenibilidad.

## 3. Conceptos base

**Alta disponibilidad**: capacidad de un sistema para mantenerse operativo aunque alguno de sus componentes falle.

```
Sin redundancia:              Con redundancia:

Usuario                       Usuario
  |                              |
Servidor único               Balanceador de carga
                                 |         |
                             Servidor A  Servidor B
```

- **Disponibilidad**: ¿el sistema sigue funcionando si algo falla?
- **Escalabilidad**: ¿el sistema puede atender más carga si aumenta la demanda?

Una arquitectura puede ser escalable pero no altamente disponible (todo en una sola zona), o altamente disponible pero no escalar automáticamente (sin Auto Scaling). Este laboratorio se enfoca en **alta disponibilidad con balanceo de carga**.

- **Load Balancer (ALB)**: distribuye las solicitudes HTTP/HTTPS entre varios servidores y usa health checks para enviar tráfico solo a los targets saludables.
- **Target Group**: conjunto de destinos (instancias EC2) a los que el balanceador envía tráfico, registrados con un protocolo y puerto.
- **Health Check**: verificación periódica (ej. `GET /health`) que determina si una instancia está saludable y puede recibir tráfico.

## 4. Arquitectura

```
Usuario / Navegador / curl
            |
Application Load Balancer (DNS público)
            |
      -------------
      |           |
  EC2 Web A    EC2 Web B
  AZ 1         AZ 2
  Apache HTTPD Apache HTTPD
```

Cada instancia sirve una página distinta con: nombre de instancia, Instance ID, zona de disponibilidad y estado del servicio, para comprobar visualmente a cuál instancia enruta el balanceador.

## 5. Restricciones del laboratorio

Por trabajar en AWS Academy sin perfiles IAM personalizados, se evita:

- Crear roles IAM ni asignar instance profiles.
- Usar CloudWatch Agent o SSM Session Manager.
- Usar Auto Scaling de forma obligatoria.
- Usar infraestructura como código que requiera permisos avanzados.

Todo se realiza desde la consola web de AWS con scripts de inicialización (User Data) en EC2.

## 6. Servicios AWS utilizados

EC2, Application Load Balancer, Target Groups, Security Groups, VPC predeterminada y subnets públicas existentes. No se crea una VPC desde cero.

## 7. Consideraciones de costo

- Usar instancias `t2.micro` o `t3.micro`.
- Eliminar el Load Balancer, las instancias EC2 y los Target Groups al finalizar.
- No dejar recursos activos después de clase (el ALB también consume créditos).

## 8. Guía de implementación

### 8.1 Preparación

Ingresar a AWS Academy Learner Lab, abrir la consola de AWS y verificar la región asignada (ej. `us-east-1`, `us-west-2`). No cambiar de región durante el laboratorio.

### 8.2 Security Groups

**`sg-alb-ha`** (Load Balancer) — tráfico HTTP desde Internet:

| Tipo | Protocolo | Puerto | Origen |
|---|---|---|---|
| HTTP | TCP | 80 | 0.0.0.0/0 |

**`sg-ec2-ha`** (instancias EC2) — tráfico HTTP únicamente desde el ALB:

| Tipo | Protocolo | Puerto | Origen |
|---|---|---|---|
| HTTP | TCP | 80 | sg-alb-ha |
| SSH (opcional) | TCP | 22 | My IP |

### 8.3 Instancias EC2

Crear dos instancias (`web-ha-a` en AZ1 y `web-ha-b` en AZ2), ambas con:

- AMI: Amazon Linux 2023
- Instance type: `t2.micro` o `t3.micro`
- Subnet pública, Auto-assign public IP: Enable
- Security Group: `sg-ec2-ha`
- IAM instance profile: `None`

En **Advanced details → User data** de cada instancia, un script instala Apache HTTPD, obtiene el `INSTANCE_ID` y la `AZ` vía metadata (IMDSv2), genera una página HTML identificando la instancia y crea el endpoint de salud:

```bash
#!/bin/bash
dnf update -y
dnf install -y httpd
systemctl enable httpd
systemctl start httpd

TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)

cat <<EOF > /var/www/html/index.html
... página HTML con Instance ID, AZ y estado ...
EOF

echo "OK" > /var/www/html/health
```

Los scripts completos para la instancia A y la instancia B están en la guía de laboratorio (carpeta de referencia del proyecto).

Verificar que ambas instancias estén en `Running` con `2/2 status checks passed`, y probar en el navegador:

```
http://IP_PUBLICA_INSTANCIA_A
http://IP_PUBLICA_INSTANCIA_B
http://IP_PUBLICA_INSTANCIA_A/health   -> OK
http://IP_PUBLICA_INSTANCIA_B/health   -> OK
```

### 8.4 Target Group

`EC2 → Target Groups → Create target group`

| Parámetro | Valor |
|---|---|
| Target type | Instances |
| Name | `tg-ha-web` |
| Protocol / Port | HTTP / 80 |
| VPC | VPC predeterminada |
| Protocol version | HTTP1 |
| Health check path | `/health` |
| Healthy / Unhealthy threshold | 2 / 2 |
| Timeout / Interval | 5s / 15s |
| Success codes | 200 |

Registrar `web-ha-a` y `web-ha-b` como targets.

### 8.5 Application Load Balancer

`EC2 → Load Balancers → Create load balancer → Application Load Balancer`

| Parámetro | Valor |
|---|---|
| Name | `alb-ha-web` |
| Scheme | Internet-facing |
| IP address type | IPv4 |
| VPC / AZs | VPC predeterminada, al menos 2 AZs con subnets públicas |
| Security Group | `sg-alb-ha` |
| Listener | HTTP 80 → forward a `tg-ha-web` |

### 8.6 Verificación

En `Target Groups → tg-ha-web → Targets`, ambas instancias deben quedar `Healthy`. Si alguna aparece `Unhealthy`, revisar: instancia encendida, Apache activo, ruta `/health` respondiendo, Security Group de EC2 permitiendo tráfico desde `sg-alb-ha`, y puerto 80 en el Target Group.

### 8.7 Prueba del balanceo

Copiar el DNS del ALB (ej. `alb-ha-web-123456.us-east-1.elb.amazonaws.com`) y abrirlo en el navegador varias veces, o probar desde terminal:

```bash
for i in {1..10}; do
  curl -s http://DNS_DEL_LOAD_BALANCER | grep "Instancia"
done
```

Se debe observar alternancia entre las respuestas de la instancia A y la instancia B.

## 9. Simulación de falla y recuperación

1. Detener `web-ha-a` (`EC2 → Instance state → Stop instance`).
2. En `Target Groups → tg-ha-web → Targets`, la instancia A pasa a `Unhealthy` o deja de recibir tráfico.
3. El DNS del ALB debe seguir respondiendo (solo desde la instancia B).
4. Iniciar nuevamente `web-ha-a` y esperar a que pase los status checks y vuelva a `Healthy` en el Target Group.
5. Verificar que el balanceador vuelve a distribuir tráfico entre ambas instancias.

Esta arquitectura mejora la disponibilidad, pero **no crea instancias nuevas automáticamente** si una falla de forma permanente. Para recuperación automática se requeriría un Launch Template + Auto Scaling Group integrado con el ALB (ver sección de extensión opcional en la guía), lo cual depende de los permisos disponibles en AWS Academy.

## 10. Actividades y entregables

La guía incluye actividades de análisis (balanceo, falla, recuperación) y una propuesta de mejora hacia producción (recuperación automática, instancias privadas, HTTPS, logs/métricas, despliegues sin caída, base de datos altamente disponible). El **reto final** consiste en un documento con diagrama de arquitectura, capturas de instancias/Target Group/ALB, evidencia de respuesta de cada instancia, evidencia de falla simulada y análisis arquitectónico.

### Evidencia

<img width="1722" height="866" alt="image" src="https://github.com/user-attachments/assets/3f22953a-bf67-45bf-b423-8450c9561d85" />

<img width="1741" height="1002" alt="image" src="https://github.com/user-attachments/assets/3a1145aa-51ec-4575-94cd-50befbf93aad" />

<img width="1683" height="976" alt="image" src="https://github.com/user-attachments/assets/60bb3495-e7df-41b2-b700-852b2077c4a2" />

<img width="1828" height="952" alt="image" src="https://github.com/user-attachments/assets/439e1693-a4fc-4478-ab12-fed5ac4115ef" />

<img width="1846" height="477" alt="image" src="https://github.com/user-attachments/assets/27d64935-a2b5-4516-bdf4-9573656b6a0f" />

<img width="1526" height="333" alt="image" src="https://github.com/user-attachments/assets/771af5ea-03d8-434a-95ac-6cb8caf9c530" />

<img width="967" height="713" alt="image" src="https://github.com/user-attachments/assets/51f33614-2596-4c17-8916-fc8d878afaae" />

<img width="852" height="462" alt="image" src="https://github.com/user-attachments/assets/2f510196-03c4-431d-82ba-4101d551abd1" />

<img width="1010" height="482" alt="image" src="https://github.com/user-attachments/assets/96123dd0-0456-432c-bba7-067761a2c2ce" />

<img width="637" height="221" alt="image" src="https://github.com/user-attachments/assets/293c32a3-61ec-418d-b18f-260ef0d2d5f3" />

<img width="652" height="305" alt="image" src="https://github.com/user-attachments/assets/14832337-a01a-4ae1-8330-fecc787860d4" />

<img width="1551" height="543" alt="image" src="https://github.com/user-attachments/assets/c69d6e06-30f5-4a81-b587-5b91b319c085" />

<img width="1536" height="682" alt="image" src="https://github.com/user-attachments/assets/d16842a2-b5e5-4f86-b221-3e2fcacc1705" />

<img width="1031" height="496" alt="image" src="https://github.com/user-attachments/assets/a222ed5b-85e3-4ce8-8768-2017756fdffb" />
















## 11. Limpieza de recursos

Al finalizar, eliminar en este orden para evitar consumo innecesario de créditos:

1. Application Load Balancer
2. Target Group
3. Instancias EC2
4. Security Groups creados

## 12. Conclusión

La alta disponibilidad no depende únicamente de "tener más servidores", sino de diseñar correctamente: redundancia, balanceo de carga, health checks, distribución por zonas, seguridad de red, recuperación ante fallos, monitoreo y limpieza operativa.
