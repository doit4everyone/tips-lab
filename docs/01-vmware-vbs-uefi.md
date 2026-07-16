---
title: "VMware Workstation bloqué par VBS/UEFI | DoIt4Everyone"
description: "Débloquer la virtualisation imbriquée sur un PC Secured-core récent avec le DG Readiness Tool et le mécanisme SecConfig.efi."
lang: fr
---
<style>
  header, footer { display: none !important; }
  .wrapper {
    max-width: 900px !important;
    margin: 0 auto !important;
    float: none !important;
    position: relative !important;
    padding: 40px 20px !important;
    font-family: "Helvetica Neue", Helvetica, Arial, sans-serif !important;
    font-size: 1.1em !important;
  }
  section {
    width: 100% !important;
    float: none !important;
    margin: 0 !important;
  }
  h1, h2 { text-align: center; }
  table { width: 100%; display: table; margin: 20px 0; }
</style>

# 🔓 VMware Workstation bloqué par VBS/UEFI

> [← Retour à l'index](../)

## Le symptôme

Vous venez de recevoir un PC neuf, souvent une machine « pro » ou « Secured-core » (HP Elite, Dell Latitude/Precision, Lenovo ThinkPad récents). Vous installez VMware Workstation, et l'installateur lui-même vous prévient, dès l'écran « Configuration compatible » :

> *« VMware Workstation Pro peut utiliser son propre hyperviseur ou la plate-forme Windows Hypervisor. Le programme d'installation a détecté que Hyper-V ou la protection des périphériques/informations d'identification est activée sur l'hôte, de sorte que les machines virtuelles s'exécuteront à l'aide de la plateforme Windows Hypervisor. La virtualisation imbriquée ne sera pas disponible. »*

Ensuite, si vous continuez et tentez de démarrer une VM avec la virtualisation imbriquée cochée dans ses réglages, un second message apparaît :

> *« Intel VT-x/EPT virtualisé n'est pas pris en charge sur cette plate-forme. Continuer sans Intel VT-x/EPT virtualisé ? »*

Ces deux messages ne sont pas des bugs d'affichage, ils décrivent exactement ce qui se passe : VMware a détecté qu'un autre hyperviseur (celui de Windows) occupe déjà le processeur, et bascule donc en mode dégradé (Windows Hypervisor Platform), qui ne peut pas exposer la virtualisation imbriquée à la VM.

Premier réflexe : vous allez dans `optionalfeatures`, vous décochez Hyper-V, la Plateforme de machine virtuelle, vous lancez `bcdedit /set hypervisorlaunchtype off`, vous redémarrez. **Rien ne change.** C'est là que commence, pour beaucoup, plusieurs heures de recherche frustrante.

---

## La fausse piste : tout ce qui touche à Windows ne sert à rien

Sur un PC neuf de ce type, le blocage ne vient presque jamais de Windows lui-même. Il vient du **firmware UEFI**.

Depuis Windows 11 22H2, et de façon renforcée avec 24H2, la Sécurité basée sur la virtualisation (VBS) et Credential Guard sont activés par défaut sur le matériel compatible (Secure Boot, TPM 2.0, VT-x/VT-d, SLAT). Sur les gammes « Secured-core », le fournisseur va plus loin : il active VBS avec un **verrou UEFI** (« Enabled with UEFI lock »). Concrètement, une variable est écrite directement dans le firmware, et cette variable force Windows à relancer VBS à chaque démarrage, **quoi que vous fassiez dans le registre ou avec bcdedit**.

Résultat : vous pouvez avoir `hypervisorlaunchtype Off` et toutes les clés de registre à zéro, VBS tournera quand même. C'est exactement pour ça que les tutoriels de 2020 qui expliquent de « décocher Hyper-V dans les fonctionnalités Windows » ne fonctionnent plus sur ce type de machine : ils traitent un symptôme qui n'est plus la cause.

Deux pièges classiques qui font perdre du temps à ce stade :
- Chercher une case à cocher liée à la gestion à distance du constructeur (type gestion cloud OEM). Ça peut ressembler à une piste sérieuse, mais ce n'est généralement pas la cause réelle.
- Se fier aux indicateurs Windows classiques (`Win32_DeviceGuard` en WMI, `msinfo32`) pour valider un changement. Ces indicateurs sont connus pour rester incohérents ou mis en cache pendant plusieurs redémarrages. Ne bouclez pas dessus indéfiniment, ils ne sont pas fiables en cours de manipulation.

---

## D'abord, éliminez le cas simple : la méthode registre suffit-elle ?

Le verrou UEFI n'est pas systématique : VBS peut être activé « with UEFI lock » ou « without UEFI lock » selon la configuration posée par l'OEM ou par GPO. Avant de sortir l'artillerie lourde, testez la méthode légère :

```powershell
REG ADD "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v "LsaCfgFlags" /t REG_DWORD /d 0 /f
REG ADD "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v "EnableVirtualizationBasedSecurity" /t REG_DWORD /d 0 /f
```

Redémarrez, puis vérifiez :

```powershell
Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard | Select-Object VirtualizationBasedSecurityStatus
```

Si vous obtenez `0`, c'est réglé, vous n'êtes pas dans le cas verrouillé, inutile d'aller plus loin.

**Si `VirtualizationBasedSecurityStatus` reste à `2` malgré un registre entièrement remis à zéro et un reboot**, c'est la signature exacte du verrou UEFI : la configuration Windows dit « désactivé », mais le firmware réimpose VBS à chaque démarrage, en dehors de tout contrôle logiciel. C'est uniquement dans ce cas de figure que la suite de cette note s'applique.

---

## La solution : le DG Readiness Tool, et une confirmation physique au boot

L'outil officiel Microsoft pour ce cas précis s'appelle **Device Guard and Credential Guard hardware readiness tool** (`DG_Readiness_Tool.ps1`), toujours en version 3.6 au moment d'écrire ces lignes, disponible sur le Download Center de Microsoft. Sa fiche annonce une compatibilité Windows 10 / Server 2016, mais il reste la référence fonctionnelle pour désactiver VBS sur Windows 11 24H2 et Windows Server 2025 : le mécanisme qu'il actionne (la variable EFI) n'a pas changé.

En PowerShell, en administrateur :

```powershell
Unblock-File .\DG_Readiness_Tool_v3.6.ps1
Set-ExecutionPolicy -Scope Process Bypass -Force
.\DG_Readiness_Tool_v3.6.ps1 -Disable
```

Le script va afficher quelques erreurs sans gravité (clé de registre introuvable, fichier `SIPolicy.p7b` absent, échec de désactivation d'un rôle Hyper-V non installé). Ce sont des artefacts liés au fait que l'outil cible une architecture Windows plus ancienne : ignorez-les.

**L'étape que presque tout le monde rate** : redémarrez la machine. Au tout premier boot, le firmware va afficher un écran de confirmation, souvent une touche à presser (F3 ou une autre selon le fabricant), parfois deux écrans successifs si VBS et Credential Guard sont tous les deux verrouillés. C'est un garde-fou volontaire : la levée d'un verrou UEFI ne peut pas se faire à distance ou par script seul, elle exige une présence physique devant la machine. Sans cette confirmation, rien ne change, même si le script a semblé « réussir ».

Une fois cette confirmation validée, vérifiez :

```powershell
Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard | Select-Object VirtualizationBasedSecurityStatus
```

`0` confirme que VBS est bien désactivé. VMware Workstation peut désormais basculer en mode hyperviseur natif et exposer VT-x/EPT au guest.

---

## Ce qui se passe réellement sous le capot

Le script PowerShell n'écrit jamais directement dans le firmware, il n'en a pas la capacité. Ce qu'il fait est plus subtil : il prépare le **prochain démarrage** pour déléguer cette tâche à un petit programme EFI signé Microsoft, `SecConfig.efi`, présent dans `System32`.

La séquence, en résumé :

1. Il monte la partition système EFI (normalement invisible) sur une lettre libre.
2. Il copie `SecConfig.efi` sur cette partition, dans `\EFI\Microsoft\Boot\`.
3. Il crée une entrée de démarrage BCD dédiée, pointant vers ce binaire, et la place en tête de la séquence de boot :
   ```
   bcdedit /create "{0cb3b571-2f2e-4343-a879-d86a476d7215}" /d "DebugTool" /application osloader
   bcdedit /set "{0cb3b571-2f2e-4343-a879-d86a476d7215}" path \EFI\Microsoft\Boot\SecConfig.efi
   bcdedit /set "{bootmgr}" bootsequence "{0cb3b571-2f2e-4343-a879-d86a476d7215}"
   bcdedit /set "{0cb3b571-2f2e-4343-a879-d86a476d7215}" loadoptions DISABLE-LSA-ISO,DISABLE-VBS
   ```
4. Il démonte la partition et rend la main.

Au redémarrage suivant, le Boot Manager charge `SecConfig.efi` **avant** Windows, en pré-OS. C'est ce binaire, et lui seul, qui affiche l'écran de confirmation et qui, une fois votre validation donnée, écrit réellement la variable EFI persistante demandant le retrait du verrou. Le script PowerShell n'a fait que poser ce binaire sur le chemin de démarrage : c'est lui qui dialogue avec le firmware, pas PowerShell.

Un point de vigilance à connaître si cette méthode ne suffisait pas sur une autre machine : il arrive que la désactivation via `DG_Readiness.ps1` seule ne tienne pas, parce que des Code Integrity Policies présentes dans la partition ESP continuent d'imposer VBS indépendamment du verrou. Dans ce cas, `CiTool.exe -lp` permet de lister les politiques actives et de vérifier si une politique VBS reste appliquée malgré tout.

---

## Pour aller plus loin : éviter le verrou dès le départ

Le fichier `ReadMe.txt` livré avec l'outil documente aussi une méthode pour activer VBS/Credential Guard **sans** poser de verrou UEFI, via la clé de registre `Locked` à `0` sous `HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard`. Utile à connaître si vous provisionnez vous-même des postes de laboratoire et voulez éviter de retomber dans ce piège à chaque réinitialisation.

---

## À retenir

| Point | Détail |
|-------|--------|
| Cause réelle | Verrou UEFI VBS/Credential Guard, pas un réglage Windows |
| Méthodes inefficaces | `bcdedit`, registre seul, décochage Hyper-V/optionalfeatures |
| Méthode qui fonctionne | DG Readiness Tool `-Disable` + confirmation physique au boot |
| Piège à éviter | Se fier aux indicateurs WMI (`Win32_DeviceGuard`), souvent trompeurs en cours de manipulation |
| Outil | Toujours en version 3.6 (juillet 2024), fiche Microsoft non mise à jour mais fonctionnel sur 24H2/Server 2025 |

Si cette note vous a fait gagner les quatre heures qu'il aura fallu pour arriver à cette conclusion, elle aura rempli son rôle.

---

> ℹ️ *Testé sur Windows 11 24H2, Windows Server 2025, VMware Workstation Pro, DG Readiness Tool v3.6*

---

[← Retour à l'index](../)
