---
title: pdata和isys的联系
date: 2018-06-01 18:00:00
categories:
- Linux
- intel
- crl
---
## pci_device
```c
static void ipu4_quirk(struct pci_dev *pci_dev)
{
	pci_dev->dev.platform_data = &pdata;
}

DECLARE_PCI_FIXUP_EARLY(PCI_VENDOR_ID_INTEL, INTEL_IPU4_HW_BXT_P,
			ipu4_quirk);
```
<!--more-->
## pci_driver
```c
static const struct pci_device_id intel_ipu4_pci_tbl[] = {
	{PCI_DEVICE(PCI_VENDOR_ID_INTEL, INTEL_IPU4_HW_BXT_B0)},
	{PCI_DEVICE(PCI_VENDOR_ID_INTEL, INTEL_IPU4_HW_BXT_P)},
	{0,}
};

static struct pci_driver intel_ipu4_pci_driver = {
	.name = INTEL_IPU4_NAME,
	.id_table = intel_ipu4_pci_tbl,
	.probe = intel_ipu4_pci_probe,
};
```

## match
通过**PCI_VENDOR_ID_INTEL, INTEL_IPU4_HW_BXT_P**来进行match，match成功的话会调用ipu4_quirk以及intel_ipu4_pci_probe

## intel_ipu4_pci_probe
### intel_ipu4_bus_device
注册了name = INTEL_IPU4_ISYS_NAME的device
```c
static int intel_ipu4_pci_probe(struct pci_dev *pdev,
			     const struct pci_device_id *id)
{
	isp->isys = intel_ipu4_isys_init(
			pdev, &isp->isys_iommu->dev, &isp->isys_iommu->dev,
			isys_base, isys_ipdata,
			pdev->dev.platform_data,
			0, is_intel_ipu_hw_fpga(isp) ?
			INTEL_IPU4_ISYS_TYPE_INTEL_IPU4_FPGA :
			INTEL_IPU4_ISYS_TYPE_INTEL_IPU4);
		return intel_ipu4_bus_add_device(pdev, parent, pdata, iommu, NULL,
					 INTEL_IPU4_ISYS_NAME, nr);
}
```

### intel_ipu4_bus_driver
```c
static struct intel_ipu4_bus_driver isys_driver = {
	.probe = isys_probe,
	.wanted = INTEL_IPU4_ISYS_NAME,
	.drv = {
		.name = INTEL_IPU4_ISYS_NAME,
	},
};
```

**INTEL_IPU4_ISYS_NAME**匹配调用**isys_probe**

**即pdata和isys_probe将会有一定的联系**
