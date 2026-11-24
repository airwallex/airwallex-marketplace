# Airwallex Embedded KYC Component: Optional Theming

> **Overview & scenarios**: see [scenarios.md](scenarios.md)

The embedded KYC component supports a `theme` option for advanced customization of look and feel. It is a **hidden field**: the `@airwallex/components-sdk` `KycElementOptions` type declares it as `theme?: Record<string, unknown>` marked `@hidden` ("Contact your Account Manager for details"), so it does not appear in the rendered public docs. Include it when creating the KYC element only if the user has specific brand color or typography requirements, and adjust the values accordingly.

> **Note**: Because the field is `@hidden` and freeform, brand theming may need to be enabled for your account by your Airwallex Account Manager (see the [embedded-KYC customization FAQ](https://www.airwallex.com/docs/connected-accounts/onboarding/kyc-and-onboarding/embedded-kyc-component)). Confirm activation before relying on specific palette/typography values.

## Table of Contents

- [Theme option](#theme-option)

---

## Theme option

```javascript
const element = await createElement('kyc', {
  theme: {
      "palette": {
        "primary": {
          "5": "#FAFBFB",
          "10": "#F6F7F8",
          "20": "#E8EAED",
          "30": "#D7DBE0",
          "40": "#B0B6BF",
          "50": "#868E98",
          "60": "#6C747F",
          "70": "#545B63",
          "80": "#42474D",
          "90": "#2F3237",
          "100": "#1A1D21"
        },
        "secondary": {
          "5": "#FFEFEF",
          "10": "#FFE0E0",
          "20": "#FDC2C2",
          "30": "#FFADAD",
          "40": "#FF776D",
          "50": "#FF4F42",
          "60": "#F50C04",
          "70": "#E00700",
          "80": "#C20303",
          "90": "#990000"
        },
        "error": {
          "5": "#FFEFEF",
          "10": "#FFE0E0",
          "20": "#FDC2C2",
          "30": "#FFADAD",
          "40": "#FF776D",
          "50": "#FF4F42",
          "60": "#F50C04",
          "70": "#E00700",
          "80": "#C20303",
          "90": "#990000"
        },
        "success": {
          "5": "#FFEFEF",
          "10": "#FFE0E0",
          "20": "#FDC2C2",
          "30": "#FFADAD",
          "40": "#FF776D",
          "50": "#FF4F42",
          "60": "#F50C04",
          "70": "#E00700",
          "80": "#C20303",
          "90": "#990000"
        },
        "warning": {
          "5": "#FFEFEF",
          "10": "#FFE0E0",
          "20": "#FDC2C2",
          "30": "#FFADAD",
          "40": "#FF776D",
          "50": "#FF4F42",
          "60": "#F50C04",
          "70": "#E00700",
          "80": "#C20303",
          "90": "#990000"
        },
        "informational": {
          "5": "#FFEFEF",
          "10": "#FFE0E0",
          "20": "#FDC2C2",
          "30": "#FFADAD",
          "40": "#FF776D",
          "50": "#FF4F42",
          "60": "#F50C04",
          "70": "#E00700",
          "80": "#C20303",
          "90": "#990000"
        },
        "gradients": {
          "primary": [],
          "secondary": [],
          "tertiary": [],
          "quaternary": [],
          "quinary": [],
          "spectrumPrimary": [],
          "spectrumSecondary": []
        }
      },
      "components": {
        "tag": {
          "colors": {
            "background": {
              "initial": "#D7DBE0",
              "primary": "#D0CDFF",
              "success": "#E0F7E7",
              "warning": "#FFECAD",
              "danger": "#FFADAD"
            },
            "text": { "initial": "#1A1D21" }
          }
        },
        "card": {
          "colors": {
            "background": { "initial": "#fff", "hover": "#fff" },
            "boxShadow": {
              "initial": "0 0 16px 0 rgba(0, 0, 0, 0.08)",
              "hover": "0 0 16px 0 rgba(0, 0, 0, 0.08)"
            },
            "border": { "initial": "#E8EAED" },
            "foreground": { "initial": "inherit", "hover": "#4F00D6" }
          }
        },
        "table": {
          "colors": {
            "headerBackground": { "hover": "#DFDEFF" },
            "rowText": { "initial": "#1A1D21" },
            "rowBorder": { "initial": "#E8EAED" },
            "rowBackground": { "hover": "#F6F7F8" },
            "paginationButtonIcon": { "disabled": "#B0B6BF", "initial": "#1A1D21" },
            "paginationText": { "initial": "#42474D" }
          }
        },
        "alert": {
          "colors": {
            "background": {
              "success": "#EFFBF3",
              "informational": "#F0F9FF",
              "warning": "#FFFBEF",
              "critical": "#FFEFEF"
            },
            "iconBackground": {
              "success": "#08AF61",
              "informational": "#0EA5E9",
              "warning": "#FF8E3C",
              "critical": "#FF4F42"
            },
            "border": {
              "success": "#83DD83",
              "warning": "#FFD014",
              "critical": "#FF776D",
              "informational": "#38BDF8"
            },
            "link": {
              "text": { "initial": "#1A1D21", "active": "#2F3237" },
              "textDecoration": {
                "hover": "#2F3237",
                "active": "#E8EAED",
                "initial": "#B0B6BF"
              },
              "outline": { "focus": "#1A1D21" }
            },
            "button": {
              "text": { "initial": "#42474D" },
              "border": { "initial": "rgba(0, 0, 0, 0.3)" },
              "background": { "hover": "rgba(255, 255, 255, 0.3)" }
            },
            "closeButton": {
              "text": { "initial": "#868E98" },
              "border": { "initial": "rgba(0, 0, 0, 0.3)" },
              "background": { "hover": "rgba(255, 255, 255, 0.3)" }
            }
          }
        },
        "button": {
          "colors": {
            "primary": {
              "background": { "initial": "#612FFF" },
              "foreground": { "initial": "#fff" }
            },
            "secondary": {
              "background": {
                "initial": "#fff",
                "hover": "#F0EFFF",
                "active": "#D0CDFF",
                "focus": "#F0EFFF"
              },
              "foreground": { "initial": "#612FFF" },
              "boxShadow": "#DFDEFF"
            },
            "danger": {
              "background": { "initial": "#FF4F42" },
              "foreground": { "initial": "#fff" }
            },
            "success": {
              "background": { "initial": "#34BACC" },
              "foreground": { "initial": "#fff" }
            },
            "link": {
              "background": { "active": "#DFDEFF", "hover": "#F0EFFF" },
              "foreground": { "initial": "#1A1D21" }
            },
            "default": {
              "background": { "initial": "#fff" },
              "foreground": { "initial": "#612FFF" }
            },
            "disabled": {
              "background": { "initial": "#E8EAED" },
              "foreground": { "initial": "#B0B6BF", "hover": "#B0B6BF" }
            }
          }
        },
        "dialog": {
          "colors": {
            "confirmDialogIcon": {
              "success": "#08AF61",
              "warning": "#FF8E3C",
              "critical": "#FF4F42"
            },
            "headerText": { "initial": "#1A1D21" },
            "bodyText": { "initial": "#6C747F" }
          }
        },
        "select": {
          "colors": {
            "control": {
              "focus": "#612FFF",
              "error": "#FF4F42",
              "initial": "#E8EAED"
            },
            "controlBackground": { "initial": "#FCFCFD", "disabled": "#F6F7F8" },
            "indicator": { "initial": "#545B63", "disabled": "#B0B6BF" },
            "placeholder": { "initial": "#868E98", "selected": "#1A1D21" },
            "option": {
              "initial": "#42474D",
              "focus": "#612FFF",
              "disabled": "#B0B6BF"
            },
            "optionBackground": {
              "initial": "#fff",
              "focus": "#F0EFFF",
              "active": "#D0CDFF",
              "disabled": "#fff"
            },
            "singleValue": { "disabled": "#B0B6BF" },
            "multiValueBackground": { "initial": "#E8EAED" },
            "multiValueRemove": { "initial": "#1A1D21", "hover": "#1A1D21" },
            "multiValueRemoveBackground": { "hover": "#B0B6BF" }
          }
        },
        "spinner": {
          "colors": {
            "stop": { "initial": "#FF8E3C" },
            "start": { "initial": "#FF4F42" }
          }
        },
        "taskFlow": {
          "colors": {
            "headerProgressBar": { "initial": "#FF8E3C" },
            "stepTickIcon": { "initial": "#08AF61" },
            "stepHighlight": { "initial": "#FF4F42" },
            "stepIndicator": { "initial": "#E8EAED" },
            "navigationArrowIcon": { "initial": "#D7DBE0", "active": "#FF8E3C" },
            "navigationTickIcon": { "initial": "#08AF61" },
            "navigationHighlight": { "initial": "#FF4F42" }
          }
        },
        "progressTracker": {
          "colors": {
            "barValue": {
              "background": {
                "default": "#775CFF",
                "success": "#08AF61",
                "warning": "#FF8E3C",
                "critical": "#FF4F42"
              }
            },
            "barBody": {
              "background": {
                "default": "#F0EFFF",
                "success": "#E0F7E7",
                "warning": "#FFF8E0",
                "critical": "#FFEDE0"
              }
            }
          }
        },
        "timeline": {
          "color": {
            "default": "#B0B6BF",
            "success": "#08AF61",
            "informational": "#545B63",
            "warning": "#D68100",
            "critical": "#E00700",
            "active": "#612FFF"
          },
          "connector": {
            "horizontal": {
              "completed": "linear-gradient(-90deg, #08AF61 40%, #08AF61 100%)",
              "current": "linear-gradient(-90deg, #612FFF 40%, #08AF61 100%)",
              "review": "linear-gradient(-90deg, #612FFF 40%, #08AF61 100%)",
              "rejected": "linear-gradient(-90deg, #E00700 40%, #08AF61 100%)"
            },
            "vertical": {
              "completed": "linear-gradient(0deg, #08AF61 20%, #08AF61 100%)",
              "current": "linear-gradient(0deg, #612FFF 20%, #08AF61 100%)",
              "review": "linear-gradient(0deg, #612FFF 20%, #08AF61 100%)",
              "rejected": "linear-gradient(0deg, #E00700 20%, #08AF61 100%)"
            }
          },
          "colors": {
            "state": {
              "initial": "#B0B6BF",
              "success": "#08AF61",
              "informational": "#545B63",
              "warning": "#D68100",
              "critical": "#E00700",
              "active": "#612FFF"
            }
          }
        },
        "textArea": {
          "colors": {
            "border": {
              "focus": "#612FFF",
              "warning": "#FFD014",
              "initial": "#E8EAED",
              "error": "#FF4F42"
            },
            "background": {
              "initial": "#FCFCFD",
              "hover": "#fff",
              "disabled": "#F6F7F8"
            },
            "text": {
              "initial": "#A9A9AE",
              "disabled": "#868E98",
              "error": "#FF4F42"
            }
          }
        },
        "dropdown": {
          "colors": {
            "menuItemText": { "initial": "#545B63", "hover": "#612FFF" },
            "menuItemBackground": {
              "hover": "#F0EFFF",
              "focus": "#F0EFFF",
              "active": "#DFDEFF"
            }
          }
        },
        "textInput": {
          "colors": {
            "border": {
              "initial": "#E8EAED",
              "focus": "#612FFF",
              "error": "#FF4F42"
            },
            "text": { "initial": "#1A1D21", "disabled": "#868E98" },
            "placeholder": { "initial": "#868E98" },
            "background": {
              "initial": "#FCFCFD",
              "hover": "#fff",
              "disabled": "#F6F7F8"
            }
          }
        },
        "datePicker": {
          "colors": {
            "buttonText": { "initial": "#1A1D21", "disabled": "#B0B6BF" },
            "buttonBackground": { "selected": "#612FFF", "hover": "#612FFF" },
            "today": { "initial": "#612FFF" },
            "tableText": { "initial": "#868E98" }
          }
        },
        "radioInput": {
          "colors": {
            "border": {
              "initial": "#B0B6BF",
              "disabled": "#B0B6BF",
              "checked": "#612FFF",
              "focus": "#612FFF",
              "hover": "#612FFF"
            },
            "background": { "initial": "#fff", "disabled": "#E8EAED" }
          }
        },
        "checkboxInput": {
          "colors": {
            "border": {
              "initial": "#B0B6BF",
              "focus": "#612FFF",
              "error": "#FF4F42",
              "checked": "#612FFF",
              "disabled": "#B0B6BF"
            },
            "background": { "disabled": "#E8EAED" }
          }
        },
        "multiSelect": {
          "colors": {
            "border": { "initial": "#DFDEFF", "focus": "#612FFF" },
            "option": { "initial": "#42474D", "selected": "#612FFF" },
            "optionBackground": { "hover": "#F0EFFF", "selected": "#F0EFFF" }
          }
        }
      },
      "typography": {
        "title": {
          "100": {
            "fontFamily": "AxLLCircular, Helvetica, Arial, sans-serif",
            "color": "#1A1D21",
            "fontWeight": 700,
            "lineHeight": 1.25,
            "fontSize": 40,
            "@media (max-width: 720px)": { "fontSize": 24 }
          }
        },
        "headline": {
          "100": {
            "fontFamily": "AxLLCircular, Helvetica, Arial, sans-serif",
            "color": "#1A1D21",
            "fontWeight": 700,
            "lineHeight": 1.25,
            "fontSize": 24
          },
          "200": {
            "fontFamily": "AxLLCircular, Helvetica, Arial, sans-serif",
            "color": "#1A1D21",
            "fontWeight": 700,
            "lineHeight": 1.25,
            "fontSize": 18
          },
          "300": {
            "fontFamily": "AxLLCircular, Helvetica, Arial, sans-serif",
            "color": "#1A1D21",
            "fontWeight": 700,
            "lineHeight": 1.25,
            "fontSize": 16
          },
          "400": {
            "fontFamily": "AxLLCircular, Helvetica, Arial, sans-serif",
            "color": "#1A1D21",
            "fontWeight": 700,
            "lineHeight": 1.25,
            "fontSize": 14
          },
          "500": {
            "fontFamily": "AxLLCircular, Helvetica, Arial, sans-serif",
            "color": "#1A1D21",
            "fontWeight": 500,
            "lineHeight": 1.25,
            "fontSize": 14
          },
          "600": {
            "fontFamily": "AxLLCircular, Helvetica, Arial, sans-serif",
            "color": "#1A1D21",
            "fontWeight": 500,
            "lineHeight": 1.25,
            "fontSize": 13,
            "textTransform": "uppercase"
          }
        },
        "body": {
          "100": {
            "fontFamily": "AxLLCircular, Helvetica, Arial, sans-serif",
            "fontSize": 14,
            "fontWeight": 400,
            "lineHeight": 1.5,
            "color": "#42474D"
          },
          "100.Subtle": {
            "fontFamily": "AxLLCircular, Helvetica, Arial, sans-serif",
            "fontSize": 14,
            "fontWeight": 400,
            "lineHeight": 1.5,
            "color": "#6C747F"
          },
          "100.Disabled": {
            "fontFamily": "AxLLCircular, Helvetica, Arial, sans-serif",
            "fontSize": 14,
            "fontWeight": 400,
            "lineHeight": 1.5,
            "color": "#B0B6BF"
          },
          "100.Bold": {
            "fontFamily": "AxLLCircular, Helvetica, Arial, sans-serif",
            "fontSize": 14,
            "fontWeight": 500,
            "lineHeight": 1.5,
            "color": "#42474D"
          }
        },
        "tabPanel": {
          "inactive": {
            "fontFamily": "AxLLCircular, Helvetica, Arial, sans-serif",
            "fontWeight": 400,
            "color": "#42474D",
            "lineHeight": 1.25,
            "fontSize": 16
          }
        },
        "label": {
          "100": {
            "fontFamily": "AxLLCircular, Helvetica, Arial, sans-serif",
            "fontSize": 14,
            "fontWeight": 400,
            "lineHeight": 1.5,
            "color": "#42474D"
          }
        },
        "hint": {
          "100": {
            "fontFamily": "AxLLCircular, Helvetica, Arial, sans-serif",
            "fontSize": 12,
            "fontWeight": 400,
            "lineHeight": 1.5,
            "color": "#6C747F"
          }
        },
        "error": {
          "100": {
            "fontFamily": "AxLLCircular, Helvetica, Arial, sans-serif",
            "fontSize": 12,
            "fontWeight": 400,
            "lineHeight": 1.5,
            "color": "#FF4F42"
          }
        },
        "textLink": {
          "fontWeight": 500,
          "fontSize": "inherit",
          "fontFamily": "inherit",
          "colors": {
            "disabled": "#B0B6BF",
            "initial": "#612FFF",
            "active": "#B3AEFF"
          }
        },
        "fontFace": {
          "customFontSource": [
            {
              "fontFamily": "Ubuntu",
              "fontStyle": "normal",
              "fontWeight": 400,
              "fontDisplay": "swap",
              "src": "url('https://fonts.gstatic.com/s/ubuntu/v20/4iCs6KVjbNBYlgoKfw72nU6AFw.woff2') format('woff2')"
            },
            {
              "fontFamily": "Ubuntu",
              "fontStyle": "normal",
              "fontWeight": 600,
              "fontDisplay": "swap",
              "src": "url('https://fonts.gstatic.com/s/ubuntu/v20/4iCs6KVjbNBYlgoKfw72nU6AFw.woff2') format('woff2')"
            }
          ]
        },
        "fontFamily": "Ubuntu"
      }
  }
});
```
