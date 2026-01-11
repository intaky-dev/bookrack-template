# Covenant FWCorp - Identity & Access Management

Covenant book para gestión de identidad y acceso usando **Keycloak + Vault** para el cluster FWCorp.

## Estructura

```
covenant-fwcorp/
├── index.yaml                          # Configuración global del realm "fwcorp"
├── README.md                           # Esta documentación
│
├── realm/                              # Configuración a nivel de realm
│   └── client-scopes/
│       └── custom-roles.yaml          # OIDC scopes para mapeo de roles
│
├── covenant/                           # Chapters (equipos/organizaciones)
│   ├── platform/                       # Platform Engineering
│   │   ├── index.yaml                 # Roles: platform-admin, platform-engineer, sre
│   │   ├── admins/                    # Administradores de plataforma
│   │   │   └── alice-admin.yaml
│   │   └── engineers/                 # Platform engineers
│   │       └── bob-engineer.yaml
│   │
│   ├── developers/                     # Development Team
│   │   ├── index.yaml                 # Roles: senior-developer, developer, junior-developer
│   │   ├── backend/                   # Backend developers
│   │   │   └── charlie-backend.yaml
│   │   └── frontend/                  # Frontend developers
│   │       └── diana-frontend.yaml
│   │
│   ├── devops/                        # DevOps Team
│   │   ├── index.yaml                 # Roles: devops-lead, devops-engineer, automation-engineer
│   │   ├── ci-cd/                     # CI/CD engineers
│   │   │   └── edward-cicd.yaml
│   │   └── infrastructure/            # Infrastructure as Code
│   │       └── frank-iac.yaml
│   │
│   └── operations/                    # Operations Team
│       ├── index.yaml                 # Roles: ops-manager, ops-engineer, support-engineer
│       ├── support/                   # Support engineers
│       │   └── grace-support.yaml
│       └── monitoring/                # Monitoring & on-call
│           └── henry-monitor.yaml
│
└── conventions/                        # Post-provisioning automation
    └── post-provisioning/
        └── keycloak-password.yaml     # Auto-genera passwords y configura Keycloak

```

## Realm Configuration

- **Realm name**: `fwcorp`
- **Email domain**: `fwcorp.io`
- **Password policy**: Min 12 chars, uppercase, lowercase, digits, special chars
- **Token lifetimes**:
  - Access token: 15 minutes
  - SSO session idle: 30 minutes
  - SSO session max: 10 hours

## Global Roles

Roles disponibles en todo el realm:

| Role | Descripción |
|------|-------------|
| `user` | Base user role (todos los usuarios) |
| `readonly` | Read-only access global |
| `full` | Full administrative access |
| `org-admin` | Organization administrator (composite: full + user) |
| `security-admin` | Security & compliance administrator |

## Chapters (Equipos)

### Platform (Plataforma)
**Roles**:
- `platform-admin`: Administrador con acceso completo a infraestructura
- `platform-engineer`: Engineer con acceso a K8s y provisioning
- `sre`: Site Reliability Engineer
- `infra-viewer`: Read-only infrastructure access

**Subdirectorios**:
- `admins/`: Platform administrators
- `engineers/`: Platform engineers

### Developers (Desarrollo)
**Roles**:
- `senior-developer`: Senior developer con privilegios elevados
- `developer`: Developer estándar
- `junior-developer`: Junior developer con acceso limitado
- `code-reviewer`: Code reviewer

**Subdirectorios**:
- `backend/`: Backend developers
- `frontend/`: Frontend developers

### DevOps
**Roles**:
- `devops-lead`: DevOps lead con acceso completo a CI/CD
- `devops-engineer`: DevOps engineer
- `automation-engineer`: Automation engineer
- `pipeline-viewer`: Read-only pipeline access

**Subdirectorios**:
- `ci-cd/`: CI/CD engineers
- `infrastructure/`: Infrastructure as Code

### Operations (Operaciones)
**Roles**:
- `ops-manager`: Operations manager
- `ops-engineer`: Operations engineer
- `support-engineer`: Support engineer
- `on-call`: On-call engineer
- `monitoring-viewer`: Read-only monitoring access

**Subdirectorios**:
- `support/`: Support engineers
- `monitoring/`: Monitoring & on-call

## Agregar Usuarios

### Formato del archivo de usuario

```yaml
# covenant/<chapter>/<subdir>/<nombre>.yaml
firstName: John
lastName: Doe
status: active
emailVerified: true

# Roles asignados al usuario
realmRoles:
  - developer
  - user

# Grupos se asignan automáticamente basado en la ruta:
# - <chapter>/<subdir>
# - <chapter>
```

### Auto-generación de campos

El sistema auto-genera:
- **Username**: `firstName.lastName` (ej: `john.doe`)
- **Email**: `firstName.lastName@fwcorp.io` (ej: `john.doe@fwcorp.io`)
- **Grupos**: Basado en la ruta del archivo

### Ejemplo: Agregar un nuevo developer

Crear archivo en `covenant/developers/backend/new-dev.yaml`:

```yaml
firstName: Jane
lastName: Smith
status: active
emailVerified: true

realmRoles:
  - senior-developer
  - code-reviewer
  - user
```

Esto generará:
- Username: `jane.smith`
- Email: `jane.smith@fwcorp.io`
- Grupos: `developers/backend`, `developers`
- Roles: `senior-developer`, `code-reviewer`, `user`

## Post-Provisioning

### keycloak-password.yaml

Script automatizado que:
1. Genera password aleatorio cumpliendo policy
2. Configura password en Keycloak vía REST API
3. Marca como `temporary=true` (fuerza cambio en primer login)
4. Guarda password en Secret de Kubernetes
5. Configura `requiredActions: UPDATE_PASSWORD`

**Recuperar password inicial**:
```bash
kubectl get secret -n keycloak covenant-initial-password-<username> \
  -o jsonpath='{.data.keycloak-password}' | base64 -d
```

## Deployment

### Testing (desde kast-system/)

```bash
# Test todos los chapters
make test-covenant-all-chapters BOOK=covenant-fwcorp

# Test chapter específico
make test-covenant-chapter BOOK=covenant-fwcorp CHAPTER=platform
```

### Deploy con ArgoCD

El librarian generará:
1. **Main Application** (sin chapterFilter):
   - KeycloakRealm
   - Roles globales
   - ApplicationSet para chapters

2. **Per-Chapter Applications** (con chapterFilter):
   - KeycloakUser (por cada archivo .yaml)
   - KeycloakGroup (por cada subdirectorio)
   - Jobs de post-provisioning

## Customización

### Cambiar dominio de email

Editar `index.yaml`:
```yaml
realm:
  emailDomain: tu-dominio.com
```

### Agregar nuevos roles globales

Editar `index.yaml` > `roles`:
```yaml
roles:
  tu-nuevo-rol:
    description: "Descripción del rol"
    composite: false
```

### Agregar roles a un chapter

Editar `covenant/<chapter>/index.yaml` > `roles`:
```yaml
roles:
  nuevo-rol-chapter:
    description: "Rol específico del chapter"
    composite: false
```

### Crear nuevo chapter

```bash
mkdir -p covenant/nuevo-chapter/subteam
```

Crear `covenant/nuevo-chapter/index.yaml`:
```yaml
name: nuevo-chapter
description: "Descripción del chapter"

roles:
  rol-especifico:
    description: "Rol del chapter"
    composite: false

memberDefaults:
  status: active
  emailVerified: true
  realmRoles:
    - user

realmRoles:
  - user
```

## Referencias

- [CLAUDE.md](../../../../CLAUDE.md) - Guía completa del sistema kast
- [Covenant Testing](../../../../CLAUDE.md#covenant-testing) - Testing de covenant books
- [covenant-tyl](../../../proto-the-yaml-life/bookrack/covenant-tyl/) - Ejemplo de referencia
