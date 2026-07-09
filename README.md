# 📸 SYSPREP - Procedimiento de Captura y Restauración de Imágenes de Máquinas Virtuales en Azure

Este manual describe el proceso estándar para ejecutar **Sysprep** en una máquina virtual (VM), capturar su imagen y restaurarla como una nueva instancia en Microsoft Azure.

---

## 1. ℹ️ Información de la Máquina Virtual de Origen

Antes de comenzar, registre los datos de la VM que servirá como origen para la imagen.

| Campo | Valor (Ejemplo) |
| :--- | :--- |
| **Grupo de Recursos** | `rg-erp-nonprod-chilecentral-01` |
| **Nombre de la VM** | `qa-erpmngsvc01` |
| **Tamaño** | Estándar B4ms (4 vCPU, 16 GiB de memoria) |
| **Disco (LUN, Nombre, Caché)** | `1 / qa-erpmngsvc01_DataDisk-SSD Premium / SSD LRS / Lectura-Escritura` |
| **Red (Vnet / Subred)** | `vnet-servers-lan-nonprod-chilecentral-01` / `snet-servers-lan-nonprod-chilecentral-01` |
| **Dirección IP Privada** | `192.168.25.199` |

---

## 2. ⚙️ Ejecutar Sysprep en la VM Origen

1.  **Inicie la máquina virtual** desde el Portal de Azure.
<br>
2.  Conéctese a ella utilizando **Azure Bastion** con las credenciales del administrador local.

    > **Advertencia:** La contraseña del ejemplo (`_9J5N1VZ2hJp2_`) es ilustrativa. Utilice siempre credenciales seguras y almacénelas en un lugar seguro como Azure Key Vault.
<br>
3.  **Gestión del usuario administrador local:**
    *   **Si el usuario no existe:** Cree un nuevo administrador local ejecutando el siguiente script en el portal (VM > Ejecutar comando > PowerShell):

        ```powershell
        $NewLocalAdmin = "Administrator"
        $Password = (ConvertTo-SecureString -AsPlainText "_9J5N1VZ2hJp2_" -Force)
        New-LocalUser "$NewLocalAdmin" -Password $Password -FullName "$NewLocalAdmin" -Description "Administrador local"
        Add-LocalGroupMember -Group "Administrators" -Member "$NewLocalAdmin"
        ```
    *   **Si el usuario ya existe:** Restablezca la contraseña del administrador desde el portal (VM > Menú > Restablecer contraseña).
<br>
4.  **Ejecute Sysprep:** Abra un símbolo del sistema como Administrador y ejecute los siguientes comandos:

    ```batch
    cd %windir%\system32\sysprep
    sysprep /generalize /shutdown /mode:vm
    ```
    > **Nota:** La VM se apagará automáticamente al finalizar el proceso. Este paso generaliza la VM, eliminando información específica del equipo (SID, nombre de host, etc.).

---

## 3. 🏷️ Marcar la VM como Generalizada

Una vez que la VM esté apagada, debe marcarla como "Generalizada" en Azure para poder crear una imagen a partir de ella.

1.  Abra una sesión de **PowerShell** (local o en Azure Cloud Shell).
<br>
2.  Conéctese a su suscripción de Azure y ejecute:

    ```powershell
    Connect-AzAccount
    Set-AzContext -SubscriptionId "SUBSCRIPTION_ID"  # Reemplace con su ID de suscripción
    Get-AzContext  # Verificar el contexto

    $vmName = "qa-erpmngsvc01"
    $rgName = "rg-erp-nonprod-chilecentral-01"

    Set-AzVm -ResourceGroupName $rgName -Name $vmName -Generalized
    ```
---

## 4. 📸 Capturar la Imagen de la VM

El proceso de captura libera los recursos de la VM original (discos, NIC, etc.) para crear una imagen reutilizable.

1.  **Elimine los recursos antiguos** de la VM original, ya no son necesarios (discos, tarjetas de red).

2.  **Cree una nueva VM a partir de la imagen** utilizando el Portal de Azure, CLI o PowerShell.

    *   **Configure la nueva VM:**
        *   Use la misma **información de red** (Vnet, Subred) y **tamaño** que la VM original.
        *   **No cambie la configuración de discos.**
        *   **No incluya una dirección IP pública** a menos que sea estrictamente necesario.
        *   **Deshabilite el diagnóstico de arranque.**
        *   **Habilite la configuración de diagnóstico** para logs, seleccionando una cuenta de almacenamiento existente (ej. `stlogs**log**xxxxxx`).

    *   **Verifique el nombre de host:** Cambie el nombre de host de la nueva VM para que coincida con el estándar (ej. de `qa-erpmngsvc01` a `erpmngsvc01`).
---

## 5. 🔧 Configuración Post-Creación de la Nueva VM

Después de que la nueva VM esté en funcionamiento, realice los siguientes ajustes:

1.  **Asignar la IP Privada:** Configure la interfaz de red de la nueva VM para usar la misma dirección IP privada que tenía la VM original (`192.168.25.199` en el ejemplo).
2.  **Configurar el DNS:** Asigne o verifique los servidores DNS correctos en la configuración de la interfaz de red para garantizar la comunicación con Active Directory.
3.  **Actualizar el Grupo de Seguridad de Red (NSG):** Agregue la nueva dirección IP de la VM a las reglas del NSG asociado a la subred para permitir la comunicación con Active Directory y otros servicios.

---

## 6. 🗑️ Limpieza Posterior

Una vez que haya verificado que la nueva VM funciona correctamente, puede:

*   **Eliminar la imagen** que ya no necesite para ahorrar costos.
*   **Eliminar los recursos** de la VM original que fueron liberados.

---

## 7. ⚠️ Consideraciones Adicionales para VMs con SQL Server

Si la máquina virtual que está capturando tiene **SQL Server** instalado, debe registrar el recurso de la nueva VM con el proveedor de recursos de SQL. Ejecute el siguiente comando después de que la nueva VM esté creada:

```powershell
$vm = Get-AzVM -Name "erpmngsvc01" -ResourceGroupName "rg-erp-nonprod-chilecentral-01"
New-AzSqlVM -Name $vm.Name -ResourceGroupName $vm.ResourceGroupName -Location $vm.Location -LicenseType AHUB -SqlManagementType Full
```

---

## 📚 Glosario de Términos

| Término | Definición |
| :--- | :--- |
| **Sysprep** | Herramienta del sistema operativo Windows para generalizar una instalación, preparándola para la clonación o captura de imagen. |
| **Azure Bastion** | Servicio de Azure que permite conexiones RDP/SSH seguras a VMs directamente desde el portal, sin exponer puertos públicos. |
| **Azure Resource Manager (ARM)** | Capa de gestión de Azure que permite desplegar y gestionar recursos de forma programática. |
| **Grupo de Seguridad de Red (NSG)** | Filtro de tráfico de red que permite o deniega el tráfico hacia y desde los recursos de Azure en una red virtual. |
| **OpenID Connect (OIDC)** | Protocolo de autenticación utilizado para Workload Identity Federation, un método seguro para autenticar flujos de trabajo CI/CD en la nube. |
| **Azure Virtual Machine (VM)** | Servicio de Azure que permite crear y gestionar máquinas virtuales bajo demanda. |
| **Azure Image** | Recurso de Azure que contiene un disco duro virtual (VHD) que puede utilizarse para crear nuevas máquinas virtuales con el mismo sistema operativo y configuración. |
| **Azure Key Vault** | Servicio de Azure para almacenar y gestionar secretos, claves y certificados de forma segura. |

---
Mario Fribla
***Ingeniero Cloud***
