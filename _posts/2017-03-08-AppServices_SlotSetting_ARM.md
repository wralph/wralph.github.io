---
title: Specifying slot settings for Azure AppServices using ARM templates
date:   2017-03-08 15:00:00 +0100
tags: Azure AppService SlotSetting
---

## Specifying slot settings for Azure AppServices using ARM templates

When thinking about AppServices within Azure, there might be the requirement to specify slot settings for specific application settings. This basically means, that a specific application setting is only used for a specific slot. It will become available after swapping to the new production slot.
See the [documentation](https://docs.microsoft.com/en-us/azure/app-service-web/web-sites-staged-publishing#configuration-for-deployment-slots) for more details how this can be useful for you. 

In this particular case, the slot setting should be used for 3 settings.

![Settings]({{ site.url }}/assets/images/slot_settings.png)


### ARM config

Of course this can be achieved by just clicking within the portal. Though, it might be more efficient to have this as part of the ARM template used to deploy the environment. The following snipped shows how this can be achieved. A resource is added to the actual website resource. Important is to specify the resoure id where this settings depend on. The properties allow you to specify the **appSettingNames** that will be slot specific settings.

```json
{
      "name": "[concat(variables('WebSiteName'), copyIndex())]",
      "copy": {
        "count": "[length(parameters('WebLocations'))]",
        "name": "WebAppLoop"
      },
      "type": "Microsoft.Web/sites",
      "location": "[parameters('WebLocations')[copyIndex()]]",
      "apiVersion": "2015-08-01",
      "dependsOn": [
        "AppSvcLoop",
        "RedisLoop",
        "[concat('Microsoft.Sql/servers/', variables('sqlserverName'),'/databases/', variables('SqlDatabaseName'))]"
      ],
      "tags": {
        "[concat('hidden-related:', resourceGroup().id, '/providers/Microsoft.Web/serverfarms/', variables('AppPlanName'), copyIndex())]": "Resource",
        "displayName": "WebSite"
      },
      "properties": {
        "name": "[concat(variables('WebSiteName'), copyIndex())]",
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms/', concat(variables('AppPlanName'), copyIndex()))]"
      },
      "resources": [
         {
          "apiVersion": "2015-08-01",
          "name": "slotconfignames",
          "type": "config",
          "dependsOn": [
            "[resourceId('Microsoft.Web/Sites', concat(variables('WebSiteName'), copyIndex()))]"
          ],
          "properties": {
            "appSettingNames": [
              "ClientId",
              "TenantId",
              "Domain"
            ]
          }
        }
      ]      
    }
```

That's it. 
Enjoy
