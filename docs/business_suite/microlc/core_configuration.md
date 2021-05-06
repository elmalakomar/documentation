---
id: core_configuration
title: Core configuration
sidebar_label: Core configuration
---

To compose the application, `microlc` needs to consume a configuration at runtime. When loaded, the
[fe-container](overview.md#front-end-container) performs a GET request to `/api/v1/microlc/configuration` expecting in
response the configuration in JSON format. This endpoint is always called, even in parallel if necessary.

## Configuration structure

The configuration is a JSON object with two root properties which defines the theming of the application, and the
characteristics of the plugins to render.

### theming

- _type_: object;
- _required_: `true`;
- _description_: contains the theming information. This property can be used to brand the application and to customize
  the user experience.

See the [theming parameters](#theming-parameters) section for details.

### plugins

- _type_: array;
- _required_: `true`;
- _description_: the list of the plugins to render. It contains information on how to integrate the plugins in `microlc`.

Each element of this array is an object correspondent to a plugin. The structure of the object is detailed in the
[plugin parameters](#plugin-parameters) section.

### analytics

- _type_: object;
- _required_: `false`;
- _description_: contains the analytics information. This property can be used to instantiate [Google Analytics](https://analytics.google.com/)
  previous user acceptance.

For the details of its content, see the [analytics parameters](#analytics-parameters) section.

## Theming parameters

This bit of the configuration object enables the customization of some part of the user experience as well as the
branding of the application.

### header

- _type_: object;
- _required_: `false`;
- _description_: contains the properties that are usually applied in the HTML `header` tag.

### pageTitle

- _type_: string;
- _required_: `false`;
- _description_: the string that will be displayed on the browser tab;
- _default_: `Microlc`.

### favicon

- _type_: string;
- _required_: `false`;
- _description_: the url of the icon that will be displayed on the browser tab;
- _default_: Mia-Platform logo.

### logo

- _type_: object;
- _required_: `true`;
- _description_: contains the information needed to display the company logo.

### url_light_image

- _type_: string;
- _required_: `true`;
- _description_: the url of the logo image for light theme.

### url_dark_image

- _type_: string;
- _required_: `false`;
- _description_: the url of the logo image for dark theme.

### navigation_url

- _type_: string;
- _required_: `false`;
- _description_: the url of the site in which the user is redirected if the logo is clicked.

### alt

- _type_: string;
- _required_: `true`;
- _description_: the alternative text to display if the logo is not found;
- _default_: `Logo`.

### variables

- _type_: object;
- _required_: `true`;
- _description_: contains the variables that will be used to brand the application overriding the default style.

### primaryColor

- _type_: string;
- _required_: `false`;
- _description_: the primary color of the application, accepted values are 3, 6, or 8 digits Hex and [CSS color keywords](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value#color_keywords).  
  Its value is applied to the `--microlc-primary-color` [`css variables`](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties),
  and is used to calculate the **89% tint** stored in the `--microlc-primary-color-tint-89` variable;
- _default_: `#1890FF`.

### menuLocation

- _type_: string;
- _required_: `false`;
- _description_: the location on the page in which the menu is located;
- _default_: `sideBar`.

## Plugin parameters

The information regarding the plugins to embed in the application are contained in these object.

### id

- _type_: string;
- _required_: `true`;
- _description_: the unique identifier of the plugin.

### aclExpression

- _type_: string;
- _required_: `false`;
- _description_: expression to evaluate the users that can access the plugin (i.e. `groups.admin && groups.ceo`).

### label

- _type_: string;
- _required_: `true`;
- _description_: the label visualized in the side menu.

### icon

- _type_: string;
- _required_: `false`;
- _description_: the icon visualized in the side menu. The supported icons are the
  [Font Awesome free](https://fontawesome.com/icons?d=gallery&p=2&m=free) ones. You have to specify all the needed
  classes;
- _example_: `fas fa-tag`;
- _default_: no icon.

### order

- _type_: integer;
- _required_: `false`;
- _description_: the position in the side menu;
- _default_: `0`.

### integrationMode

- _type_: string;
- _enum_: `hfer`, `qiankun`, `iframe`;
- _required_: `true`;
- _description_: the way in which the plugin is integrated in `microlc`, see [Plugin configuration](plugin_configuration.md) section for mode details.

### pluginRoute

- _type_: string;
- _required_: `true` for `integrationMode` of type `qiankun` or `iframe`;
- _description_: the path of the main application on which the plugin is rendered.

### pluginUrl

- _type_: string;
- _required_: `true` for `integrationMode` of type `qiankun` or `iframe`;
- _description_: the entry point of the plugin (i.e., where the plugin is deployed).

### props

- _type_: object;
- _required_: `false`;
- _description_: contains the properties injected during the boostrap of a plugin rendered with `qiankun`.

### externalLink

- _type_: object;
- _required_: `true` for `integrationMode` of type `href`;
- _description_: contains the details about the href integration.

### url

- _type_: string;
- _required_: `true`;
- _description_: the url of the external application.

### sameWindow

- _type_: boolean;
- _required_: `true`;
- _description_: states if the link should be opened in a new window;
- _default_: `false`

## Analytics parameters

### privacyLink

- _type_: string;
- _required_: `true`;
- _description_: link used to redirect the user to the privacy policy

### disclaimer

- _type_: string;
- _required_: `true`;
- _description_: disclaimer showed to the user, generally used to inform him about the cookies that will be used

### gtmId

- _type_: string;
- _required_: `true`;
- _description_: container id for [Google Analytics](https://analytics.google.com/);
- _example_: `GTM-XXXXXX`

## Example

```json
{
  "theming": {
    "header": {
      "pageTitle": "Mia-Platform",
      "favicon": "https://favicon-url.com"
    },
    "logo": {
      "url_light_image": "https://logo-url.com/light",
      "url_dark_image": "https://logo-url.com/dark",
      "alt": "My awesome logo"
    },
    "variables": {
      "primaryColor": "red"
    }
  },
  "analytics": {
    "privacyLink": "https://policies.google.com/",
    "disclaimer": "We use analytics cookies for...",
    "gtmId": "GTM-XXXXXX"
  },
  "plugins": [
    {
      "id": "plugin-1",
      "aclExpression": "groups.admin || groups.superadmin",
      "label": "Plugin number 1",
      "icon": "fas fa-rocket",
      "order": 0,
      "integrationMode": "qiankun",
      "pluginRoute": "/myAwesomePlugin",
      "pluginUrl": "https://plugin-url.com",
      "props": {}
    },
    {
      "id": "plugin-2",
      "allowedGroups": [],
      "label": "Plugin number 2",
      "icon": "fas fa-tag",
      "order": 1,
      "integrationMode": "href",
      "externalLink": {
        "url": "https://external-site.com",
        "sameWindow": false
      }
    }
  ]
}
```