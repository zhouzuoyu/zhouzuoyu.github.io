---
title: pdata和isys的联系(二)
date: 2018-06-01 18:00:00
categories:
- Linux
- intel
- crl
---
## pdev->dev.platform_data
```c
static struct intel_ipu4_isys_subdev_pdata pdata = {
	.subdevs = (struct intel_ipu4_isys_subdev_info *[]) {
		&ov13858_crl_sd,
		...
	},
};

static void ipu4_quirk(struct pci_dev *pci_dev)
{
	pci_dev->dev.platform_data = &pdata;
}
```
<!--more-->
## pdata->spdata
```c
static int intel_ipu4_pci_probe(struct pci_dev *pdev,
			     const struct pci_device_id *id)
	isp->isys = intel_ipu4_isys_init(
			pdev, &isp->isys_iommu->dev, &isp->isys_iommu->dev,
			isys_base, isys_ipdata,
			pdev->dev.platform_data,	// FIXME
			0, is_intel_ipu_hw_fpga(isp) ?
			INTEL_IPU4_ISYS_TYPE_INTEL_IPU4_FPGA :
			INTEL_IPU4_ISYS_TYPE_INTEL_IPU4);
		pdata->spdata = spdata;			// FIXME
		return intel_ipu4_bus_add_device(pdev, parent, pdata, iommu, NULL,
						 INTEL_IPU4_ISYS_NAME, nr);
			adev = kzalloc(sizeof(*adev), GFP_KERNEL);
			adev->pdata = pdata;		// FIXME
}
```

```c
static int isys_probe(struct intel_ipu4_bus_device *adev)
{
	isys->pdata = adev->pdata;

	rval = isys_register_devices(isys);
		isys_register_ext_subdevs(isys);
			struct intel_ipu4_isys_subdev_pdata *spdata = isys->pdata->spdata;	// FIXME
			for (sd_info = spdata->subdevs; *sd_info; sd_info++)
				isys_register_ext_subdev(isys, *sd_info, false);
			
}
```
所以sd_info就指向了ov13858_crl_sd，将依次遍历pdata->subdevs。
