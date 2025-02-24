# Windows 10 - Custom Kernel Signers

## 🌟 What is Custom Kernel Signers?

Windows 10 enforces strict requirements for kernel-mode drivers. Since version 1607, all new drivers must be signed by an EV certificate trusted by Microsoft and submitted to the Windows Hardware Portal for verification. Even if a self-signed certificate is installed in Windows Certificate Store (`certlm.msc` or `certmgr.msc`), Windows 10 will refuse to load the driver unless **TestSigning mode** is enabled. This indicates that Windows 10 maintains a separate certificate store specifically for kernel-mode drivers.

**Custom Kernel Signers (CKS)** is a hidden product policy in Windows 10 (possibly introduced in version 1703) under the full name `CodeIntegrity-AllowConfigurablePolicy-CustomKernelSigners`. This feature allows users to control which certificates are trusted or denied in the kernel. However, enabling this policy might require `CodeIntegrity-AllowConfigurablePolicy` to be active as well.

By default, **CKS is disabled** on all Windows 10 editions except **Windows 10 China Government Edition**.

If the following conditions are met:

1. `CodeIntegrity-AllowConfigurablePolicy-CustomKernelSigners` is enabled (potentially requiring `CodeIntegrity-AllowConfigurablePolicy`).
2. Secure Boot is enabled.

Then, users with access to the **UEFI Platform Key (PK)** can install their own certificates into the kernel certificate store, allowing the execution of any driver signed with the specified certificate on that machine.

For more details on Windows product policies, refer to [this guide](https://www.geoffchappell.com/notes/windows/license/install.htm).

---

## 🔧 Enabling Custom Kernel Signers

### 1️⃣ Prerequisites

- **Administrator privileges** are required.
- **A temporary Windows 10 Enterprise or Education environment** is needed to execute `ConvertFrom-CIPolicy` in PowerShell, as this is not available on other editions.
- **Access to the UEFI Platform Key (PK)** is necessary.

### 2️⃣ Creating Certificates and Setting Platform Key (PK)

Follow [this guide](asset/build-your-own-pki.md) to generate the required certificates. You will obtain the following files:

```plaintext
// Self-signed root CA certificate
localhost-root-ca.der
localhost-root-ca.pfx

// Kernel-mode certificate issued by root CA
localhost-km.der
localhost-km.pfx

// UEFI Platform Key certificate issued by root CA
localhost-pk.der
localhost-pk.pfx
```

Setting the Platform Key varies depending on the firmware. Below is an example for VMware users:

#### 🔹 Setting PK in VMware

1. **Shut down your virtual machine (`TestVM`).**
2. **Delete** the `TestVM.nvram` file (this resets UEFI settings on next boot).
3. **Modify `TestVM.vmx`** by adding the following lines:

   ```plaintext
   uefi.allowAuthBypass = "TRUE"
   uefi.secureBoot.PKDefault.file0 = "localhost-pk.der"
   ```

4. **Start `TestVM`** and your PK will be set.

---

## 🛠️ Building Kernel Code-Signing Certificate Rules

Run **PowerShell as Administrator** in Windows 10 Enterprise/Education:

```powershell
New-CIPolicy -FilePath SiPolicy.xml -Level RootCertificate -ScanPath C:\windows\System32Add-SignerRule -FilePath .\SiPolicy.xml -CertificatePath .\localhost-km.der -Kernel
ConvertFrom-CIPolicy -XmlFilePath .\SiPolicy.xml -BinaryFilePath .\SiPolicy.bin
```

This generates the necessary policy rules, which can now be applied to any Windows 10 edition.

---

## 🏴‍☠️ Persisting Custom Kernel Signers

Since Windows 10 resets **CKS** within 10 minutes unless it's the **China Government Edition**, persistence is required.

1. **Sign `ckspdrv.sys` with the kernel-mode certificate:**

   ```plaintext
   signtool sign /fd sha256 /ac .\localhost-root-ca.der /f .\localhost-km.pfx /p <password> ckspdrv.sys
   ```

2. **Install and start the signed driver:**

   ```plaintext
   sc create ckspdrv binpath=%windir%\system32\drivers\ckspdrv.sys type=kernel start=auto error=normal
   sc start ckspdrv
   ```

Once done, **any driver signed with your certificate can be loaded**. 🚀

---

## 🇷🇺 Русская версия

### Что такое Custom Kernel Signers?

Windows 10 требует, чтобы драйверы ядра были подписаны сертификатом EV, доверенным Microsoft. С версии 1607 новые драйверы должны проходить проверку через Windows Hardware Portal. Даже если сертификат добавлен в хранилище сертификатов Windows (`certlm.msc`), он **не загрузится без включенного TestSigning**.

**Custom Kernel Signers (CKS)** позволяет управлять доверенными сертификатами в ядре, но требует включения `CodeIntegrity-AllowConfigurablePolicy`.

### Как включить?

1. **Получите права администратора.**
2. **Создайте сертификаты и настройте Platform Key (PK).**
3. **Настройте Secure Boot.**
4. **Создайте и примените правила подписи кода.**
5. **Обеспечьте постоянство CKS, используя подписанный драйвер.**

После этих шагов ваш драйвер будет загружаться без ограничений. 🚀

---

## 🇫🇷 Version Française

### Qu'est-ce que Custom Kernel Signers ?

Windows 10 impose des restrictions strictes sur les drivers en mode noyau. Depuis la version 1607, tous les drivers doivent être signés avec un certificat EV approuvé par Microsoft et validé via le Windows Hardware Portal. Même si un certificat auto-signé est ajouté au magasin de certificats Windows (`certlm.msc`), il **ne sera pas chargé sans le mode TestSigning**.

**Custom Kernel Signers (CKS)** est une politique permettant aux utilisateurs de choisir les certificats approuvés par le noyau, mais nécessite l’activation de `CodeIntegrity-AllowConfigurablePolicy`.

### Comment l’activer ?

1. **Obtenez des droits administrateur.**
2. **Générez des certificats et configurez le Platform Key (PK).**
3. **Activez Secure Boot.**
4. **Créez et appliquez les règles de signature du noyau.**
5. **Rendez la configuration persistante avec un driver signé.**

Après ces étapes, vous pourrez charger n'importe quel driver signé avec votre certificat ! 🚀
