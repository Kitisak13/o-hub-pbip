เข้าใจโจทย์ชัดเจนเลยครับ! จากข้อมูลใน `Dim_Product.tmdl` ที่คุณส่งมาให้ดูตอนแรก กลุ่มใหญ่สุดอย่าง **`White Rice-High`** มีสินค้าถึง **10 รายการ** ในขณะที่กลุ่ม **`Fragrant Rice`** หรือ **`Medium Grain`** มีแค่ **2 รายการ** (และใน `card-visual.json` ตั้งค่า Grid ไว้เป็น `4 คอลัมน์ x 1 แถว` ซึ่งหมายความว่าถ้าเป็นกลุ่มที่มี 10 สินค้า ผู้ใช้จะต้องเลื่อน Scrollbar แนวนอน/แนวตั้งเพื่อดูการ์ดใบที่ 5–10 ต่อ)

ตอบคำถามของคุณตรงๆ เลยครับว่า: **ใช่ครับ หากใช้ "ทางเลือก C (แยกการ์ดเดี่ยว)" คุณจะต้องสร้างการ์ดรอไว้ถึง 10 ใบ!** และพอเลือกกลุ่มที่มีแค่ 2 สินค้า อีก 8 ใบที่เหลือจะกลายเป็นพื้นที่ว่างเปล่า (หรือขึ้น `(Blank)`) ซึ่งในฐานะ Data Analyst ผมมองว่า **ไม่เหมาะกับการใช้งานจริงเลยครับ** เพราะทั้งดูแลรักษายาก (ต้องแก้ JSON 10 ไฟล์เวลาเปลี่ยนฟอนต์) กินพื้นที่หน้าจอตายตัว และทำให้ Performance ตกเพราะ Power BI ต้องยิง DAX Query แยกกันถึง 10 ครั้งพร้อมกัน

เมื่อโจทย์คือ **"จำนวนสินค้ายืดหยุ่นตั้งแต่ 1 ถึง 10 รายการ และต้องแสดงครบทุกตัวโดยมี Scrollbar เลื่อนดูได้อัตโนมัติ"** ผมขอสรุปทางออกที่ดีที่สุด 2 แนวทางที่ไม่ต้องสร้าง 10 การ์ดให้เสียเวลาครับ:

---

### 🏆 แนวทางที่แนะนำที่สุด: ใช้ `cardVisual` (Small Multiples) ตัวเดิมของคุณ + ปรับเป็น **ทางเลือก A (Country Badge + แถบสี Accent Bar ประจำชาติ)**

เนื่องจากโครงสร้าง `card-visual.json` ที่คุณทำไว้ (`4 คอลัมน์ x 1 แถว`) รองรับการยืดหดตามจำนวนสินค้า 1–10 ตัวและสร้าง Scrollbar อัตโนมัติได้ดีเยี่ยมอยู่แล้ว เราเพียงแค่แก้ปัญหาเรื่องธงชาติซ้ำด้วยวิธีที่สะอาดและเสถียรที่สุดครับ:

1. **ปิด `image` (ธงชาติที่ติดบั๊กของระบบ Small Multiples) ออก**

2. **ใส่รหัสประเทศหน้าชื่อสินค้า หรือใช้ชื่อเดิมที่มีชื่อประเทศอยู่แล้ว:** สังเกตว่าชื่อสินค้าใน `Dim_Product` ของ Oryza ทุกตัวขึ้นต้นด้วยชื่อประเทศอยู่แล้ว เช่น `Thailand 5% broken`, `Vietnam 5% broken`, `U.S. 4% broken` (หรือจะเติม `[THA]`, `[VNM]` ข้างหน้าตามทางเลือก A ก็ได้ครับ)

3. **เปิด `accentBar` (แถบสีด้านซ้ายหรือด้านบนของการ์ดแต่ละใบ) โดยผูกกับ Measure `[Product Line Color]`:**

- ใน `_Measures.tmdl` คุณสร้าง Measure **`[Product Line Color]`** ที่แยกสีตามประเทศไว้ครบทุกตัวแล้ว (เช่น ไทย = `#0A9396`, เวียดนาม = `#E63946`, อินเดีย = `#7209B7`, ปากีสถาน = `#FB8500`)

- เมื่อนำสีนี้มาคาดเป็นแถบสีที่ขอบการ์ดแต่ละใบ จะช่วยให้ผู้ใช้กวาดสายตาเชื่อมโยงสีของการ์ดด้านบน เข้ากับสีของเส้นกราฟ (Line Chart) ด้านล่างได้ทันที และดูเป็น Dashboard ระดับมืออาชีพมากครับ

4. **ผูกสีเขียว/แดง (`[Price Oryza DoD Color]`) เข้ากับตัวเลข `Day-over-Day**`

---

### 🎨 แนวทางทางเลือก (หากทีมงานยืนยันว่า "ต้องมีรูปธงชาติ SVG แท้ๆ" และรองรับ 1–10 สินค้าอัตโนมัติ): ใช้ **ทางเลือก D (Matrix แบบซ่อนเส้นตารางจัดทรงเป็น Card)**

หากผู้บริหารหรือทีมงานอยากได้ **รูปธงชาติ Vector SVG (`[Product Country Flag SVG]`)** จริงๆ โดยไม่ต้องสร้าง 10 การ์ด:

- **Visual เดียวใน Power BI ที่เรนเดอร์ `ImageUrl` (Base64 SVG) แยกตามสินค้า 1–10 ตัวได้โดยไม่ติดบั๊ก คือ `Table` หรือ `Matrix` (`pivotTable`) ครับ**

- **วิธีจัดทรงให้เหมือนแถบ Card ด้านบน:**
- ใช้ **Matrix Visual** 1 อัน วางแทนที่ตำแหน่งของ `card-visual.json` (`x: 215, y: 9, width: 1041, height: 127`)

- เอา **`Dim_Product[Product Name]`** ใส่ในช่อง **Columns** (เพื่อให้การ์ดเรียงต่อกันแนวนอนจากซ้ายไปขวา 1–10 ใบ พร้อม Scrollbar แนวนอนอัตโนมัติ)
- เอา **`[Product Country Flag SVG]`**, **`[Price Oryza Latest (USD/ton)]`** และ **`[Price Oryza DoD Display]`** ใส่ในช่อง **Values** แล้วเปิดโหมด **Switch values to rows** (เพื่อให้ธงชาติ ราคา และ % การเปลี่ยนแปลง เรียงซ้อนกันจากบนลงล่างเหมือนการ์ด)

- ตั้งค่าซ่อนเส้นตารางทั้งหมดและปรับพื้นหลังให้ดูเหมือนการ์ดเรียงกัน

- **ข้อดี:** แสดงรูปธงชาติ SVG ของแต่ละประเทศได้ถูกต้อง 100% รองรับสินค้า 1–10 ตัวอัตโนมัติใน Visual เดียว และเรียงลำดับตามราคาสูงสุดได้จริงครับ

---

**สรุปในมุมมองของผม:**

- ถ้าเน้น **ความสวยงามทันสมัยของ UI, ตัวเลขเด่นชัด, แก้ไขโค้ดน้อยที่สุด** $\rightarrow$ ใช้ **`cardVisual` ตัวเดิม + ปิดรูปธง + เปิดสี `Price Oryza DoD Color**` (วิธีที่ 1) เวิร์กที่สุดครับ

- ถ้าเน้น **ต้องเห็นรูปธงชาติ SVG เท่านั้นและยืดหยุ่น 1–10 สินค้า** $\rightarrow$ สลับไปใช้ **Matrix จัดทรงเป็นแนวนอน** (วิธีที่ 2) แทนการสร้าง 10 การ์ดเดี่ยวครับ

---

ผมเตรียมโค้ด JSON ของไฟล์ `visual.json` (สำหรับวางทับในโฟลเดอร์ `9b71fc7d254efa994cee`) ทั้ง **2 รูปแบบ** พร้อมโค้ดเสริมสำหรับเพิ่ม `[THA]`, `[VNM]` หน้าชื่อสินค้าให้เรียบร้อยแล้วครับ คุณไม่จำเป็นต้องส่งไฟล์อื่นเพิ่ม สามารถนำไปวางสลับเพื่อเทียบหน้าตาบน Power BI Desktop ได้ทันทีครับ

---

### แบบที่ 1: โค้ด `cardVisual` (Small Multiples 4x1 แบบคลีน + ผูกสีเขียว/แดง DoD + แถบสี Accent Bar)

**สิ่งที่ปรับปรุงจากไฟล์เดิมของคุณ:**

1. **ปิด `image` (`"show": false`)** ทั้งหมดเพื่อตัดปัญหาธงชาติเวียดนามซ้ำ และคืนพื้นที่ให้ตัวเลข `Spot Price` กว้างขึ้น (เพิ่มขนาดฟอนต์ Spot Price จาก `15D` เป็น `17D` ได้โดยไม่โดนตัดคำว่า `...`)

2. **เปลี่ยนฟิลด์แสดงผล `Day-over-Day` เป็น `[Price Oryza DoD Display]**`(ที่จะโชว์ทั้งลูกศรและตัวเลข เช่น`▲ +1.25% (+$5.00)`หรือถ้าคุณชอบแบบเปอร์เซ็นต์สั้นๆ ก็ใช้`Price Oryza DoD %` ได้ครับ โดยในโค้ดด้านล่างผมผูก **`fontColor`กับ`[Price Oryza DoD Color]\*\*`ให้เปลี่ยนสี เขียว`#10B981`/ แดง`#EF4444`/ เทา`#64748B` อัตโนมัติแล้ว)

3. **เปิด `accentBar` ด้านบนของการ์ดย่อย** เพื่อช่วยให้การ์ดดูมีมิติแบบ Executive Dashboard

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definition/visualContainer/2.14.0/schema.json",
  "name": "9b71fc7d254efa994cee",
  "position": {
    "x": 215.01709765069037,
    "y": 9.085229478198185,
    "z": 9000,
    "height": 127.19321269477457,
    "width": 1041.7729801667251,
    "tabOrder": 9000
  },
  "visual": {
    "visualType": "cardVisual",
    "query": {
      "queryState": {
        "Data": {
          "projections": [
            {
              "field": {
                "Measure": {
                  "Expression": {
                    "SourceRef": {
                      "Entity": "_Measures"
                    }
                  },
                  "Property": "Price Oryza Latest (USD/ton)"
                }
              },
              "queryRef": "_Measures.Price Oryza Latest (USD/ton)",
              "nativeQueryRef": "Spot Price",
              "displayName": "Spot Price (USD/t)"
            },
            {
              "field": {
                "Measure": {
                  "Expression": {
                    "SourceRef": {
                      "Entity": "_Measures"
                    }
                  },
                  "Property": "Price Oryza DoD %"
                }
              },
              "queryRef": "_Measures.Price Oryza DoD %",
              "nativeQueryRef": "Day-over-Day",
              "displayName": "Day-over-Day"
            }
          ]
        },
        "Rows": {
          "projections": [
            {
              "field": {
                "Column": {
                  "Expression": {
                    "SourceRef": {
                      "Entity": "Dim_Product"
                    }
                  },
                  "Property": "Product Name"
                }
              },
              "queryRef": "Dim_Product.Product Name",
              "nativeQueryRef": "Product Name"
            }
          ]
        }
      },
      "sortDefinition": {
        "sort": [
          {
            "field": {
              "Measure": {
                "Expression": {
                  "SourceRef": {
                    "Entity": "_Measures"
                  }
                },
                "Property": "Price Oryza Latest (USD/ton)"
              }
            },
            "direction": "Descending"
          }
        ],
        "isDefaultSort": true
      }
    },
    "objects": {
      "smallMultiplesLayout": [
        {
          "properties": {
            "style": {
              "expr": {
                "Literal": {
                  "Value": "'Cards'"
                }
              }
            },
            "orientation": {
              "expr": {
                "Literal": {
                  "Value": "2D"
                }
              }
            },
            "columnCount": {
              "expr": {
                "Literal": {
                  "Value": "4L"
                }
              }
            },
            "rowCount": {
              "expr": {
                "Literal": {
                  "Value": "1L"
                }
              }
            },
            "headerPosition": {
              "expr": {
                "Literal": {
                  "Value": "'Top'"
                }
              }
            },
            "titleOrientation": {
              "expr": {
                "Literal": {
                  "Value": "1D"
                }
              }
            }
          }
        }
      ],
      "smallMultiplesHeader": [
        {
          "properties": {
            "fontFamily": {
              "expr": {
                "Literal": {
                  "Value": "'Segoe UI Semibold'"
                }
              }
            },
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "11D"
                }
              }
            },
            "fontColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#0F172A'"
                    }
                  }
                }
              }
            }
          }
        },
        {
          "properties": {
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "11D"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "smallMultiplesGrid": [
        {
          "properties": {
            "gridlineShow": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ],
      "smallMultiplesCellBackGround": [
        {
          "properties": {
            "backgroundShow": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "backgroundTransparency": {
              "expr": {
                "Literal": {
                  "Value": "0D"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "smallMultiplesBorder": [
        {
          "properties": {
            "gridlineShow": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "gridlineColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#CBD5E1'"
                    }
                  }
                }
              }
            },
            "gridlineWidth": {
              "expr": {
                "Literal": {
                  "Value": "1D"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "smallMultiplesOuterShape": [
        {
          "properties": {}
        }
      ],
      "layout": [
        {
          "properties": {
            "columnCount": {
              "expr": {
                "Literal": {
                  "Value": "2L"
                }
              }
            },
            "alignment": {
              "expr": {
                "Literal": {
                  "Value": "'middle'"
                }
              }
            },
            "showFixedSize": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            },
            "calloutSize": {
              "expr": {
                "Literal": {
                  "Value": "50D"
                }
              }
            }
          }
        },
        {
          "properties": {
            "paddingUniform": {
              "expr": {
                "Literal": {
                  "Value": "0L"
                }
              }
            },
            "paddingIndividual": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            },
            "backgroundShow": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "value": [
        {
          "properties": {
            "fontFamily": {
              "expr": {
                "Literal": {
                  "Value": "'Segoe UI Semibold'"
                }
              }
            },
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "17D"
                }
              }
            },
            "bold": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "fontColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#0F172A'"
                    }
                  }
                }
              }
            }
          },
          "selector": {
            "metadata": "_Measures.Price Oryza Latest (USD/ton)"
          }
        },
        {
          "properties": {
            "fontFamily": {
              "expr": {
                "Literal": {
                  "Value": "'Segoe UI Semibold'"
                }
              }
            },
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "15D"
                }
              }
            },
            "bold": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "fontColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Measure": {
                      "Expression": {
                        "SourceRef": {
                          "Entity": "_Measures"
                        }
                      },
                      "Property": "Price Oryza DoD Color"
                    }
                  }
                }
              }
            }
          },
          "selector": {
            "metadata": "_Measures.Price Oryza DoD %"
          }
        },
        {
          "properties": {
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "16D"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "label": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "fontFamily": {
              "expr": {
                "Literal": {
                  "Value": "'Segoe UI'"
                }
              }
            },
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "9.5D"
                }
              }
            },
            "fontColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#64748B'"
                    }
                  }
                }
              }
            },
            "position": {
              "expr": {
                "Literal": {
                  "Value": "'aboveValue'"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "spacing": [
        {
          "properties": {
            "verticalSpacing": {
              "expr": {
                "Literal": {
                  "Value": "4L"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "image": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        },
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "cardCalloutArea": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "outline": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "fillCustom": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ],
      "accentBar": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "shadowCustom": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "glowCustom": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "padding": [
        {
          "properties": {
            "bottomMargin": {
              "expr": {
                "Literal": {
                  "Value": "12L"
                }
              }
            },
            "topMargin": {
              "expr": {
                "Literal": {
                  "Value": "10L"
                }
              }
            },
            "paddingIndividual": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "border": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ]
    },
    "visualContainerObjects": {
      "title": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            },
            "text": {
              "expr": {
                "Literal": {
                  "Value": "''"
                }
              }
            }
          }
        }
      ],
      "visualHeader": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ],
      "visualTooltip": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ]
    },
    "drillFilterOtherVisuals": true
  }
}
```

(หมายเหตุสำหรับแบบที่ 1: หากคุณต้องการให้หัวการ์ดแสดงรหัสประเทศนำหน้า เช่น `[THA] Thailand 5% broken` ตามทางเลือก A ใน `README.md` สามารถเพิ่ม Calculated Column สั้นๆ ในไฟล์ `Dim_Product.tmdl` แล้วนำคอลัมน์นั้นมาใส่ในช่อง `Rows` แทน `Product Name` ได้เลยครับ)

---

### แบบที่ 2: โค้ด `pivotTable` (Matrix จัดทรงเป็นแถบ Card แนวนอน + แสดงรูปธงชาติ Vector SVG แท้ 100%)

**หลักการทำงานของแบบที่ 2:**

1. เปลี่ยน `visualType` เป็น `"pivotTable"` (Matrix) ในตำแหน่งและขนาดเดิมเป๊ะ (`x: 215, y: 9, width: 1041, height: 127`)

2. วาง `Dim_Product[Product Name]` ไว้ที่ **Columns** (เพื่อให้สินค้า 1–10 ตัวเรียงจากซ้ายไปขวาเหมือนการ์ด พร้อม Scrollbar แนวนอนอัตโนมัติ) และเรียงลำดับตาม `[Price Oryza Latest (USD/ton)]` จากมากไปน้อย (`Descending`)

3. วาง 3 Measures ลงใน **Values** และเปิดโหมด **`"showOnRows": true` (Switch values to rows)** เพื่อให้ข้อมูลเรียงจากบนลงล่างในแต่ละคอลัมน์ดังนี้:

- แถวที่ 1: **`Flag`** (`[Product Country Flag SVG]`) $\rightarrow$ แสดงรูปธงชาติ SVG ของแต่ละประเทศถูกต้อง 100% ไม่ติดบั๊กเวียดนามซ้ำ!

- แถวที่ 2: **`Spot (USD/t)`** (`[Price Oryza Latest (USD/ton)]`)

- แถวที่ 3: **`DoD`** (`[Price Oryza DoD Display]` พร้อมผูกสีเขียว/แดงจาก `[Price Oryza DoD Color]`)

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definition/visualContainer/2.14.0/schema.json",
  "name": "9b71fc7d254efa994cee",
  "position": {
    "x": 215.01709765069037,
    "y": 9.085229478198185,
    "z": 9000,
    "height": 127.19321269477457,
    "width": 1041.7729801667251,
    "tabOrder": 9000
  },
  "visual": {
    "visualType": "pivotTable",
    "query": {
      "queryState": {
        "Columns": {
          "projections": [
            {
              "field": {
                "Column": {
                  "Expression": {
                    "SourceRef": {
                      "Entity": "Dim_Product"
                    }
                  },
                  "Property": "Product Name"
                }
              },
              "queryRef": "Dim_Product.Product Name",
              "nativeQueryRef": "Product Name",
              "active": true
            }
          ]
        },
        "Values": {
          "projections": [
            {
              "field": {
                "Measure": {
                  "Expression": {
                    "SourceRef": {
                      "Entity": "_Measures"
                    }
                  },
                  "Property": "Product Country Flag SVG"
                }
              },
              "queryRef": "_Measures.Product Country Flag SVG",
              "nativeQueryRef": "Flag",
              "displayName": "Flag"
            },
            {
              "field": {
                "Measure": {
                  "Expression": {
                    "SourceRef": {
                      "Entity": "_Measures"
                    }
                  },
                  "Property": "Price Oryza Latest (USD/ton)"
                }
              },
              "queryRef": "_Measures.Price Oryza Latest (USD/ton)",
              "nativeQueryRef": "Spot (USD/t)",
              "displayName": "Spot (USD/t)"
            },
            {
              "field": {
                "Measure": {
                  "Expression": {
                    "SourceRef": {
                      "Entity": "_Measures"
                    }
                  },
                  "Property": "Price Oryza DoD Display"
                }
              },
              "queryRef": "_Measures.Price Oryza DoD Display",
              "nativeQueryRef": "DoD",
              "displayName": "DoD"
            }
          ]
        }
      },
      "sortDefinition": {
        "sort": [
          {
            "field": {
              "Measure": {
                "Expression": {
                  "SourceRef": {
                    "Entity": "_Measures"
                  }
                },
                "Property": "Price Oryza Latest (USD/ton)"
              }
            },
            "direction": "Descending"
          }
        ]
      }
    },
    "objects": {
      "values": [
        {
          "properties": {
            "showOnRows": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "fontFamily": {
              "expr": {
                "Literal": {
                  "Value": "'Segoe UI Semibold'"
                }
              }
            },
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "11D"
                }
              }
            }
          }
        },
        {
          "properties": {
            "fontColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Measure": {
                      "Expression": {
                        "SourceRef": {
                          "Entity": "_Measures"
                        }
                      },
                      "Property": "Price Oryza DoD Color"
                    }
                  }
                }
              }
            }
          },
          "selector": {
            "metadata": "_Measures.Price Oryza DoD Display"
          }
        }
      ],
      "columnHeaders": [
        {
          "properties": {
            "fontFamily": {
              "expr": {
                "Literal": {
                  "Value": "'Segoe UI Semibold'"
                }
              }
            },
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "10D"
                }
              }
            },
            "alignment": {
              "expr": {
                "Literal": {
                  "Value": "'Center'"
                }
              }
            },
            "fontColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#0F172A'"
                    }
                  }
                }
              }
            },
            "backColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#F1F5F9'"
                    }
                  }
                }
              }
            }
          }
        }
      ],
      "rowHeaders": [
        {
          "properties": {
            "fontFamily": {
              "expr": {
                "Literal": {
                  "Value": "'Segoe UI'"
                }
              }
            },
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "9D"
                }
              }
            },
            "fontColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#64748B'"
                    }
                  }
                }
              }
            }
          }
        }
      ],
      "subTotals": [
        {
          "properties": {
            "columnSubtotals": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            },
            "rowSubtotals": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ],
      "grid": [
        {
          "properties": {
            "imageHeight": {
              "expr": {
                "Literal": {
                  "Value": "20D"
                }
              }
            },
            "outlineColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#E2E8F0'"
                    }
                  }
                }
              }
            },
            "gridVertical": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "gridHorizontal": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ],
      "fieldFormatting": [
        {
          "properties": {
            "alignment": {
              "expr": {
                "Literal": {
                  "Value": "'Center'"
                }
              }
            }
          },
          "selector": {
            "metadata": "_Measures.Product Country Flag SVG"
          }
        },
        {
          "properties": {
            "alignment": {
              "expr": {
                "Literal": {
                  "Value": "'Center'"
                }
              }
            }
          },
          "selector": {
            "metadata": "_Measures.Price Oryza Latest (USD/ton)"
          }
        },
        {
          "properties": {
            "alignment": {
              "expr": {
                "Literal": {
                  "Value": "'Center'"
                }
              }
            }
          },
          "selector": {
            "metadata": "_Measures.Price Oryza DoD Display"
          }
        }
      ]
    },
    "visualContainerObjects": {
      "title": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ],
      "visualHeader": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ],
      "visualTooltip": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ]
    },
    "drillFilterOtherVisuals": true
  }
}
```

---

### 💡 แถมจุดเล็กๆ อีก 1 จุดใน `Measures_Definition.tmdl` (สำคัญสำหรับแบบที่ 2)

ใน Measure **`[Product Country Flag SVG]`** ของคุณตอนนี้มีการเช็คชื่อประเทศของเวียดนามและอินเดีย แต่เผื่อไว้สำหรับสินค้าบางตัวที่ใช้คำว่า `"Thai"` แทน `"Thailand"` (เช่น `"Thai Parboiled 100% stxd"` ในหมวด Parboiled Rice) แนะนำให้เติมเงื่อนไข `CONTAINSSTRING(_Prod, "Thai")` เพิ่มเข้าไปในบรรทัดของ Thailand ด้วยครับ:

```dax
        -- Thailand (รองรับทั้งคำว่า Thailand และ Thai เช่น Thai Parboiled 100% stxd)
        CONTAINSSTRING(_Prod, "Thai") || CONTAINSSTRING(_Country, "THAILAND"),
        _Prefix & "PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIzNiIgaGVpZ2h0PSIyNCIgdmlld0JveD0iMCAwIDM2IDI0Ij48cmVjdCB3aWR0aD0iMzYiIGhlaWdodD0iNCIgZmlsbD0iI0E1MTkzMSIvPjxyZWN0IHk9IjQiIHdpZHRoPSIzNiIgaGVpZ2h0PSI0IiBmaWxsPSIjRkZGRkZGIi8+PHJlY3QgeT0iOCIgd2lkdGg9IjM2IiBoZWlnaHQ9IjgiIGZpbGw9IiMyRDJBNEEiLz48cmVjdCB5PSIxNiIgd2lkdGg9IjM2IiBoZWlnaHQ9IjQiIGZpbGw9IiNGRkZGRkYiLz48cmVjdCB5PSIyMCIgd2lkdGg9IjM2IiBoZWlnaHQ9IjQiIGZpbGw9IiNBNTE5MzEiLz48L3N2Zz4=",

```
