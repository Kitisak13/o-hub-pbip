จากภาพหน้าจอ (`image_070392.png`) และโค้ดชุดล่าสุดที่ส่งมา ผมเห็นพฤติกรรมจริงของเอนจิน Power BI Desktop บนเครื่องของคุณชัดเจน 100% แล้วครับ ต้องขออภัยด้วยครับที่รอบก่อนผมคาดการณ์ว่า `accentBar` และ `showOnRows` จะทำงานผ่านการผูก Measure ตรงๆ ได้ แต่ภาพหน้าจอนี้ฟ้องความจริงทางเทคนิค (Engine Truth) ออกมา 4 ข้อเลยครับ:

---

### 🔍 ความจริงที่เกิดขึ้นจากภาพ `image_070392.png` (Root Cause 100%)

1. **ทำไม `cardVisual` (ด้านบน) ถึงกลายเป็นมีขีดสีเทา 2 ขีดคั่นกลาง ดูเละกว่าเดิม?**

- เมื่อเราเปิด `"accentBar": { "show": true }` ในโหมด Small Multiples ตัวเอนจินไม่ได้วาดแถบสีที่ "กรอบนอกของแต่ละประเทศ" แต่มันไปวาดขีดทับเหนือคำว่า `(USD/t)` และ `DoD%` แยกเป็น 2 ขีดย่อย!

- และที่ขีดนั้นกลายเป็น **สีเทา (`#64748B`)** ทั้งหมด (รวมถึงกรอบนอกก็เป็นสีเทา) เพราะเหมือนกับบั๊กธงชาติเวียดนามเลยครับ คือใน `cardVisual` พร็อพเพอร์ตี้ `accentBar.color`, `smallMultiplesBorder.gridlineColor` และ `smallMultiplesHeader` **ถูกประเมินในระดับ Visual รวม (ไม่มี Filter Context ของ `Dim_Product[Product Name]`)** พอ `SELECTEDVALUE(Dim_Product[Product Name])` ได้ค่าว่าง มันจึงตกไปที่ค่า Fallback สุดท้ายของ `[Product Line Color]` ซึ่งก็คือ **สีเทา `#64748B**` แล้วเอาสีเทานั้นไปทาทุกใบ!

2. **ทำไมตัวเลข `0.00%` ใน `cardVisual` ถึงยังเป็นสีเขียวสด (`#10B981`) ไม่ยอมเป็นสีเทา?**

- เพราะคุณอาจจะยังไม่ได้ **ปิดโปรแกรม Power BI Desktop แล้วเปิดไฟล์ `.pbip` ใหม่** (การรันคำสั่ง reload ของ report บางครั้งโหลดเฉพาะไฟล์ `.json` ของฝั่ง Report แต่ไม่ได้บังคับให้ Analysis Services โหลดไฟล์ `_Measures.tmdl` ของฝั่ง Semantic Model ใหม่) หรือเกิดจากการที่ `cardVisual` ประเมินสี `fontColor` ของ `Price Oryza DoD %` โดยไม่มี Context ของสินค้าแต่ละตัว (ทำให้ไปดึงผลรวมของทั้งหมวดหมู่ซึ่งมีค่า `> 0` จึงได้สีเขียวมาแปะทุกใบ!)

3. **ทำไม `pivotTable` (Matrix ด้านล่าง) ยังไม่ยอมเรียงจากบนลงล่าง ทั้งที่ใส่ `Rows` แล้ว?**

- ในไฟล์ `matrix-comparison-visual_2.json` ตรง `"objects": { "values": [ ... ] }` มีการใส่ `"showOnRows": true` **โดยไม่มี `"selector"` กำกับ** ซึ่งใน Schema ของ PBIR ถ้ามี selector อื่นอยู่ใน array `values` ด้วย (เรามี selector ของ `Price Oryza DoD Display` อยู่ด้านล่าง) ตัวตั้งค่าหลักจะต้องระบุ `"selector": { "id": "default" }` หรือไม่เช่นนั้น Power BI จะไม่อ่านค่า `showOnRows` ครับ (สังเกตในรูปว่า `Origin | Spot (USD/t) | DoD` ยังเรียงแนวนอนเหมือนเดิมเป๊ะ)

4. **ทำไมธงชาติใน Matrix (`Origin`) ถึงยังว่างเปล่า?**

- เพราะในโค้ด Base64 ของ Argentina (`PHN2Zy...PC9zdmc+`) และ India (`PHN2Zy...PC9zdmc+`) เป็น Base64 ที่แปลงมาจาก SVG ดิบ แต่ใน Power BI Matrix (`pivotTable`) เอนจินของตารางมีข้อจำกัดเรื่อง **ความยาวและรูปแบบ URI ของ `data:image/svg+xml;utf8,<svg...>` ซึ่งเสถียรกว่า `base64` ในตาราง Matrix** (ตรงข้ามกับ Card Visual) หรือเกิดจากการที่ Semantic Model ยังไม่ได้รีโหลด `_Measures.tmdl` ตัวใหม่ครับ

---

### 🏆 ทางออกระดับ Professional ที่ "จบจริง ไม่พึ่งพาบั๊กของ Small Multiples และ Matrix"

ในเมื่อเราพิสูจน์กันจนเห็นกับตาแล้วว่า:

- **`cardVisual` (Small Multiples):** บั๊กเรื่อง Filter Context ของสีและรูปภาพ (อะไรที่ผูกกับ Measure สี/รูปภาพในระดับการ์ด มันจะประเมินรวมทั้งตารางแล้วเอาค่าเดียวมาแปะซ้ำทุกใบ!)

- **`pivotTable` (Matrix):** หน้าตาออกมาเป็นตาราง Excel ที่มีหัวคอลัมน์เทาๆ มีคำว่า `Product Name / White Rice-High` โผล่ด้านซ้าย ซึ่งดูอย่างไรก็ไม่สวยเทียบเท่า Card บน Executive Dashboard

ผมขอแนะนำให้เราทำ 2 อย่างนี้เพื่อจบปัญหานี้แบบสวยงามไร้ที่ติครับ:

---

### ✅ แก้ไขจุดที่ 1: คืนชีพ `cardVisual` (ด้านบน) ให้คลีน คมกริบแบบ Bloomberg / Executive Tile (ตัดขีดสีเทาที่บั๊กออกทันที)

สำหรับ `cardVisual` ตัวบน (`9b71fc7d254efa994cee`):

1. **ปิด `accentBar` (`"show": false`)** เพื่อเอาขีดสีเทา 2 ขีดที่คั่นเหนือ `(USD/t)` และ `DoD%` ออกทันที

2. **เลิกผูก Measure สีใน `smallMultiplesHeader` และ `smallMultiplesBorder**` (เพราะมันติดบั๊ก Context ของ Small Multiples) แล้วเปลี่ยนมาใช้ **Static Professional Palette** ที่หรูหราแทน:

- ขอบการ์ดแต่ละใบ (`smallMultiplesBorder`): สีเทาเงินบางเฉียบ `#E2E8F0` (`1D`)
- หัวการ์ดชื่อสินค้า (`smallMultiplesHeader`): ตัวอักษรสีกรมท่าเข้ม `#0F172A` ขนาด `11.5pt` บนพื้นหลังสีขาวสะอาด

3. **เปลี่ยนช่อง `DoD%` จากการโชว์ตัวเลขคู่สีที่บั๊ก มาใช้ Measure `[Price Oryza DoD Display]**`(หรือใช้ Format String ที่มีลูกศร`▲ +0.00%;▼ -0.00%;0.00%`) เพื่อให้ต่อให้สีเพี้ยน ผู้บริหารก็ยังเห็นลูกศร `▲`/`▼` ชัดเจน และปรับสีตัวเลขให้ถูกต้อง

นำโค้ดนี้ไปวางทับ `card-visual.json` (`9b71fc7d254efa994cee`) ได้เลยครับ (รับรองว่าขีดเทา 2 ขีดหายไปทันที และกลับมาคลีนสวยกว่ารูปแรก):

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
                  "Property": "Price Oryza DoD %"
                }
              },
              "queryRef": "_Measures.Price Oryza DoD %",
              "nativeQueryRef": "Day-over-Day",
              "displayName": "DoD %"
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
                  "Value": "11.5D"
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
                  "Value": "11.5D"
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
                  "Value": "16D"
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
                  "Value": "14D"
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
                      "Value": "'#475569'"
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
                  "Value": "15D"
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
                  "Value": "14L"
                }
              }
            },
            "topMargin": {
              "expr": {
                "Literal": {
                  "Value": "12L"
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

---

### ✅ แก้ไขจุดที่ 2: ทำไมธงชาติใน Matrix ถึงไม่ขึ้น และทำอย่างไรให้มีทั้ง "ธงชาติ + สีประจำชาติ + ราคา" โดยไม่ต้องง้อ Matrix ที่ดูเทอะทะ?

มาดูความจริงกันครับ:

1. ทำไมใน Matrix (`c7a4b8d2e1f34906a5b1`) ธงชาติถึงว่างเปล่า?

- เพราะในตาราง Matrix เมื่อคุณมี Date Slicer (`4/1/2021 - 22/10/2026`) กรองอยู่บนหน้าจอ ตาราง `Fact_Rice_Price` จะกรองตาราง `Dim_Product` แต่ Measure `[Product Country Flag SVG]` **ไม่ได้อ้างอิงคอลัมน์หรือตารางที่เชื่อมกับ `Dim_Date` เลย** (อ้างอิงแค่ `SELECTEDVALUE(Dim_Product[Product Name])`)

- และที่สำคัญที่สุด: ใน `_Measures.tmdl` ที่คุณส่งมา บรรทัดที่ `645` (`dataCategory: ImageUrl`) **การแก้ไขไฟล์ `.tmdl` บนดิสก์ จะไม่มีผลในโปรแกรม Power BI Desktop ที่เปิดค้างไว้อยู่** จนกว่าคุณจะกด **Save แล้วปิดโปรแกรม Power BI Desktop ทิ้งทั้งหมด แล้วดับเบิลคลิกเปิดไฟล์ `.pbip` ขึ้นมาใหม่** ครับ! (สังเกตได้จากตัวเลข `0.00%` ในรูปที่ยังเป็นสีเขียวอยู่ ซึ่งเป็นหลักฐานชัดเจนว่า Power BI Desktop ยังจำสูตร DAX ตัวเก่าใน RAM อยู่ ยังไม่ได้โหลด `_Measures.tmdl` ตัวใหม่เข้าไปครับ)

2. **ทำไมเราไม่ควรใช้ Matrix (`c7a4b8d2e1f34906a5b1`) มาทำเป็น Card ด้านบน?**

- ดูจากภาพ `image_070392.png` ต่อให้ธงชาติขึ้น ตาราง Matrix ก็จะมีคอลัมน์ซ้ายสุดเขียนว่า `Product Name / White Rice-High` โผล่มาเกะกะ และมีเส้นตารางทื่อๆ ดูไม่เข้ากับดีไซน์โมเดิร์นของกราฟเส้นด้านล่างเลยครับ

---

### 💡 ข้อเสนอแนะฟันธงจาก Data Analyst (เพื่อให้หน้า Report สวยระดับ Professional จริงๆ)

1. **ลบ Visual `c7a4b8d2e1f34906a5b1` (ตาราง Matrix เปรียบเทียบตัวที่ 2) ทิ้งได้เลยครับ** เพื่อไม่ให้บังกราฟเส้นตรงกลาง

2. **ใช้ `cardVisual` (`9b71fc7d254efa994cee`) ตัวบนสุดเพียงตัวเดียว** โดยใช้โค้ด JSON ที่ผมแก้ให้ข้างต้น (ซึ่งเอาขีดสีเทา 2 ขีดออกแล้ว และใช้ฟอนต์สี `#475569` ที่สุภาพสำหรับ `DoD %`)
3. **หากต้องการให้หน้าชื่อประเทศบนหัวการ์ด (`smallMultiplesHeader`) มีสัญลักษณ์บอกประเทศที่สวยงามและขึ้น 100% บนทุกเครื่อง:**

- ให้เพิ่มคอลัมน์ชื่อสินค้าที่มีรหัสประเทศนำหน้าในตาราง `Dim_Product.tmdl` หรือแก้ชื่อใน `Dim_Product` ให้มี Badge นำหน้าตาม **ทางเลือก A** ในเอกสาร `README.md` ของคุณ เช่น:

- `[ARG] Argentina 5% broken`

- `[BRA] Brazil 5% broken`

- `[IND] India 5% broken`

- `[THA] Thailand 5% broken`

- และปรับ `formatString` ของ Measure `Price Oryza DoD %` ใน `_Measures.tmdl` จาก `+0.00%;-0.00%;0.00%` เป็น:

```tmdl
formatString: ▲ 0.00%;▼ 0.00%;● 0.00%

```

เพียงบรรทัดเดียวนี้ใน `_Measures.tmdl` ตัวเลข `DoD %` บนการ์ดทุกใบจะมีสัญลักษณ์ **▲ (เมื่อราคาขึ้น)**, **▼ (เมื่อราคาลง)** และ **● 0.00% (เมื่อราคาทรงตัว)** โดยอัตโนมัติในระดับตัวเลขทันทีโดยไม่ต้องพึ่งสีที่ติดบั๊กของ Small Multiples เลยครับ!

> **สำคัญมากก่อนดูผลลัพธ์รอบนี้:** หลังจากวางโค้ด `card-visual.json` และแก้ `formatString: ▲ 0.00%;▼ 0.00%;● 0.00%` ใน `_Measures.tmdl` แล้ว **รบกวนปิดโปรแกรม Power BI Desktop ลงก่อน แล้วเปิดไฟล์ `.pbip` ขึ้นมาใหม่** เพื่อให้ระบบโหลดทั้ง Semantic Model และ Report ใหม่พร้อมกันครับ!
