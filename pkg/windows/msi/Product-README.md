
## Product attributes

### UpgradeCode
GUID defining the product across versions. E.g. a previous version is uninstalled during upgrade.
In other words: for update (or upgrade), Windows Installer relies on the UpgradeCode attribute of the Product tag.
Keep the same UpgradeCode GUID as long as you want the products to be upgraded by the installer.

### Id
[wixtoolset](https://wixtoolset.org/documentation/manual/v3/xsd/wix/product.html): The product code GUID for the product. Type: AutogenGuid

[MS](https://docs.microsoft.com/en-us/windows/win32/msi/product-codes): The product code is a GUID that is the principal identification of an application or product.

[MS](https://docs.microsoft.com/en-us/windows/win32/msi/productcode): This ID must vary for different versions and languages.

[MS](https://docs.microsoft.com/en-us/windows/win32/msi/changing-the-product-code): The product code must be changed if any of the following are true for the update:
- The name of the .msi file has been changed.

[MS](https://docs.microsoft.com/en-us/windows/win32/msi/major-upgrades):
A major upgrade is a comprehensive update of a product that needs a change of the ProductCode Property.
A typical major upgrade removes a previous version of an application and installs a new version.

A constant Product code GUID is (only) useful for a subsequent mst (transform).
To be safe for a major upgrade, the Id (product code GUI) is dynamic/autogenerated: * (star)

Therefore: we use dynamic/autogenerated: * (star)


## Conditions (for install)

[doc](https://wixtoolset.org/documentation/manual/v3/xsd/wix/condition.html)

[expression-syntax](https://www.firegiant.com/wix/tutorial/com-expression-syntax-miscellanea/expression-syntax)

The XML CDATA Section <![CDATA[ and ]]> is safer.

## Properties
Most important [Naming conventions](https://docs.microsoft.com/en-us/windows/win32/msi/restrictions-on-property-names):

- Public properties may be changed by the user and must be upper-case.

Logic value and checkboxes:

-  A msi property is false if and only if it is unset, undefined, missing, the empty string (msi properties are strings).
-  A checkbox is empty if and only if the relevant msi property is false.


[OS Properties](http://wixtoolset.org/documentation/manual/v3/howtos/redistributables_and_install_checks/block_install_on_os.html)

- MsiNTProductType:  1=Workstation  2=Domain controller  3=Server
- VersionNT:
  - Windows  7=601   [msdn](https://msdn.microsoft.com/library/aa370556.aspx)
  - Windows 10=603   [ms](https://support.microsoft.com/en-us/help/3202260/versionnt-value-for-windows-10-and-windows-server-2016)
- PhysicalMemory     [ms](https://docs.microsoft.com/en-us/windows/desktop/Msi/physicalmemory)




msi properties, use in custom actions:
-  DECAC = "Deferred cusmtom action in C#"
-  CADH  = "Custom action data helper"
-  The CADH helper must mention each msi property or the DECAC function will crash:
-  A DECAC that tries to use a msi property not listed in its CADH crashes.

Example:

In the CADH:

    master=[MASTER];minion_id=[MINION_ID]

In the DECAC:

    session.CustomActionData["master"]      THIS IS OK
    session.CustomActionData["mister"]      THIS WILL CRASH


### Conditional removal of lifetime data
"Lifetime data" means any change that was not installed by the msi (during the life time of the application).

When uninstalling an application, an msi only removes exactly the data it installed, unless explicit actions are taken.

Salt creates life time data which must be removed, some of it during upgrade, all of it (except configuration) during uninstall.

Wix `util:RemoveFolderEx` removes any data transaction safe, but counts an upgrade as an uninstallation.
- for salt/bin/** (mostly *.pyc) this is appropriate.
- for salt/var/** (custom grains and modules) we restrict deletion to "only on uninstall" (`REMOVE ~= "ALL"`).


### Delete minion_id file
Alternatives

https://wixtoolset.org/documentation/manual/v3/xsd/wix/removefile.html

https://stackoverflow.com/questions/7120238/wix-remove-config-file-on-install




## Sequences
An msi is no linear program.
To understand when custom actions will be executed, one must look at the condition within the tag and Before/After:

On custom action conditions:
[Common-MSI-Conditions.pdf](http://resources.flexerasoftware.com/web/pdf/archive/IS-CHS-Common-MSI-Conditions.pdf)
[ms](https://docs.microsoft.com/en-us/windows/win32/msi/property-reference)

On the upgrade custom action condition:

|  Property   |  Comment  |
| --- |  --- |
|  UPGRADINGPRODUCTCODE | does not work
|  Installed            | the product is installed per-machine or for the current user
|  Not Installed        | there is no previous version with the same UpgradeCode
|  REMOVE ~= "ALL"      | Uninstall

[Custom action introduction](https://docs.microsoft.com/en-us/archive/blogs/alexshev/from-msi-to-wix-part-5-custom-actions-introduction)

### Articles
"Installation Phases and In-Script Execution Options for Custom Actions in Windows Installer"
http://www.installsite.org/pages/en/isnews/200108/


## Standard action sequence

[Standard actions reference](https://docs.microsoft.com/en-us/windows/win32/msi/standard-actions-reference)

[Standard actions WiX default sequence](https://www.firegiant.com/wix/tutorial/events-and-actions/queueing-up/)

[coding bee on Standard actions WiX default sequence](https://codingbee.net/wix/wix-the-installation-sequence)

You get error LGHT0204 when  After or Before are wrong. Example:

    del_NSIS_DECAC is a in-script custom action.  It must be sequenced between InstallInitialize and InstallFinalize in the InstallExecuteSequence

Notes on ReadConfig_IMCAC

    Note 1:
      Problem: INSTALLDIR was not set in ReadConfig_IMCAC
      Solution:
      ReadConfig_IMCAC must not be called BEFORE FindRelatedProducts, but BEFORE MigrateFeatureStates because
      INSTALLDIR in only set in CostFinalize, which comes after FindRelatedProducts
      Maybe one could call ReadConfig_IMCAC AFTER FindRelatedProducts
    Note 2:
      ReadConfig_IMCAC is in both InstallUISequence and InstallExecuteSequence,
      but because it is declared Execute='firstSequence', it will not be repeated in InstallExecuteSequence if it has been called in InstallUISequence.


## Don't allow downgrade
http://wixtoolset.org/documentation/manual/v3/howtos/updates/major_upgrade.html


## VC++ for Python

Quote from [PythonWiki](https://wiki.python.org/moin/WindowsCompilers):
Even though Python is an interpreted language, you **may** need to install Windows C++ compilers in some cases.
For example, you will need to use them if you wish to:

- Install a non-pure Python package from sources with Pip (if there is no Wheel package provided).
- Compile a Cython or Pyrex file.

**The msi contains only required VC++ runtimes.**

The Salt-Minion requires the C++ runtime for:

- The x509 module requires M2Crypto
  - M2Crypto requiresOpenSSL
    - OpenSSL requires "vcredist 2013"/120_CRT


Microsoft provides the Visual C++ compiler.
The runtime come with Visual Studio (in `C:\Program Files (x86)\Common Files\Merge Modules`).
Merge modules (*.msm) are msi 'library' databases that can be included ('merged') into a (single) msi databases.

Which Microsoft Visual C++ compiler is needed where?

| Software                         | msm         | from Visual Studio and in "vcredist" name
|---                               |---          |---
|  (CPython 2.7)                   | VC90_CRT    | 2008
|  M2Crypto, OpenSSL               | VC120_CRT   | 2013
|  (CPython 3.5, 3.6, 3.7, 3.8)    | VC140_CRT   | 2015

The msi incorporates merge modules following this [how-to](https://wixtoolset.org/documentation/manual/v3/howtos/redistributables_and_install_checks/install_vcredist.html)


## Images
Images:

- Dimensions of images must follow [WiX rules](http://wixtoolset.org/documentation/manual/v3/wixui/wixui_customizations.html)
- WixUIDialogBmp must be transparent

Create Product-imgLeft.png from panel.bmp:

- Open paint3D:
  - new image, ..., canvas options: Transparent canvas off, Resize image with canvas NO, Width 493 Height 312
  - paste panel.bmp, move to the left, save as



## Note on Create folder

          Function win_verify_env()  in  salt/slt/utils/verify.py sets permissions on each start of the salt-minion services.
          The installer must create the folder with the same permissions, so you keep sets of permissions in sync.

          The Permission element(s) below replace any present permissions,
          except NT AUTHORITY\SYSTEM:(OI)(CI)(F), which seems to be the basis.
          Therefore, you don't need to specify User="[WIX_ACCOUNT_LOCALSYSTEM]"  GenericAll="yes"

          Use icacls to test the result:
            C:\>icacls salt
            salt BUILTIN\Administrators:(OI)(CI)(F)
                 NT AUTHORITY\SYSTEM:(OI)(CI)(F)
            ~~ read ~~
            (object inherit)(container inherit)(full access)

            C:\>icacls salt\bin\include
            salt\bin\include BUILTIN\Administrators:(I)(OI)(CI)(F)
                             NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
                             w7h64\Markus:(I)(OI)(CI)(F)
            ~~ read ~~
            (permission inherited from parent container)(object inherit)(container inherit)(full access)

          Maybe even the Administrator group full access is "basis", so there is no result of the instruction,
          I leave it for clarity, and potential future use.

## On servicePython.wxs

      Experimental. Intended to replace nssm (ssm) with the Windows service control.
           Maybe, nssm (ssm) cannot be replaced, because it indefineiy starts the salt-minion python exe over and over again,
           whereas the Windows method only starts an exe only a limited time and then stops.
           Also goto BuildDistFragment.xsl and remove python.exe
      <ComponentRef Id="servicePython" />

## Set permissions of the install folder with WixQueryOsWellKnownSID

[doc](http://wixtoolset.org/documentation/manual/v3/customactions/osinfo.html)
