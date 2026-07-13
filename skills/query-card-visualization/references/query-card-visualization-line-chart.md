# lineChart

JSON schema for the `lineChart` visualizationTemplate object. The `visualizationTemplate` value MUST be a single object matching this schema (top-level key `lineChart`). Do not write it from memory; emit exactly this shape.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "lineChart": {
      "description": "Configuration for line chart visualizations in Vega. Displays data as a series of points connected by lines, often used for time series data.",
      "type": "object",
      "properties": {
        "x": {
          "description": "The field to use for the X-axis. Determines which data field's values will be mapped to the X-axis.",
          "anyOf": [
            {
              "type": "string",
              "description": "A simple string identifier for a column in a dataset.\n\nThis is the most basic field type, representing a direct reference to a column by its ID.\n\nEXAMPLES:\n- \"timestamp\": Reference to a timestamp column\n- \"status_code\": Reference to a status code column\n- \"user_id\": Reference to a user ID column\n    "
            },
            {
              "type": "object",
              "properties": {
                "id": {
                  "type": "string",
                  "description": "The ID of the base column containing the JSON data.\n\n        This identifies the column that contains the JSON object you want to extract a value from.\n        "
                },
                "path": {
                  "type": "string",
                  "description": "A JSON path expression to extract a specific value from a JSON column.\n\nThe path can be one level deep or multiple levels deep.\nThe each level of the path must be quoted, or surrounded by brackets if it is an array index.\nIf one level deep, use the key name directly. If multiple levels deep, use dot notation to navigate.\n\nIMPORTANT: For each level of the path, it must be enclosed in quotes.\n           Do not use brackets for non numeric array indices.\n           Array indices do not need to be quoted and should use bracket notation.\n\nEXAMPLES:\n- \"name\": Extract the 'name' field from the top level of a JSON object\n- \"k8s.cluster.name\": Extract \"k8s.cluster.name\" from the top level of a JSON object. Quotes are required for keys with dots.\n- \"user\".\"email\": Extract the 'email' field from the 'user' object\n- \"items\"[0].\"id\": Extract the 'id' field from the first item in an 'items' array\n"
                }
              },
              "required": ["id", "path"],
              "description": "A field that references a value within a JSON column using a path expression.\n\nUse this when you need to access nested data within a JSON structure.\n\nEXAMPLES:\nExtract the email field from the user object in the 'data' JSON column.\n{ \"id\": \"data\", \"path\": '\"user\".\"email\"' }\n\nExtract k8s.cluster.name from the top level of a JSON object.\n{ \"id\": \"data\", \"path\": '\"k8s.cluster.name\"' } \nNotice the quotes around \"k8s.cluster.name\" to handle keys with dots.\n\nExtract service.name from the top level of a JSON object.\n{ \"labels\": \"data\", \"path\": '\"service.name\"' }\nNotice the quotes around \"service.name\" to handle keys with dots.\n    "
            },
            {
              "type": "object",
              "properties": {
                "label": {
                  "type": "string",
                  "description": "A human-readable label for the link field.\n\nThis is used for display purposes in the UI and helps identify what the link represents.\n\nEXAMPLES:\n- \"Customer\": A link to a customer record\n- \"Parent Order\": A link to a parent order\n        "
                },
                "srcFields": {
                  "type": "array",
                  "items": {
                    "anyOf": [
                      {
                        "type": "string",
                        "description": "A simple string identifier for a column in a dataset.\n\nThis is the most basic field type, representing a direct reference to a column by its ID.\n\nEXAMPLES:\n- \"timestamp\": Reference to a timestamp column\n- \"status_code\": Reference to a status code column\n- \"user_id\": Reference to a user ID column\n    "
                      },
                      {
                        "type": "object",
                        "properties": {
                          "id": {
                            "type": "string",
                            "description": "The ID of the base column containing the JSON data.\n\n        This identifies the column that contains the JSON object you want to extract a value from.\n        "
                          },
                          "path": {
                            "type": "string",
                            "description": "A JSON path expression to extract a specific value from a JSON column.\n\nThe path can be one level deep or multiple levels deep.\nThe each level of the path must be quoted, or surrounded by brackets if it is an array index.\nIf one level deep, use the key name directly. If multiple levels deep, use dot notation to navigate.\n\nIMPORTANT: For each level of the path, it must be enclosed in quotes.\n           Do not use brackets for non numeric array indices.\n           Array indices do not need to be quoted and should use bracket notation.\n\nEXAMPLES:\n- \"name\": Extract the 'name' field from the top level of a JSON object\n- \"k8s.cluster.name\": Extract \"k8s.cluster.name\" from the top level of a JSON object. Quotes are required for keys with dots.\n- \"user\".\"email\": Extract the 'email' field from the 'user' object\n- \"items\"[0].\"id\": Extract the 'id' field from the first item in an 'items' array\n"
                          }
                        },
                        "required": ["id", "path"],
                        "description": "A field that references a value within a JSON column using a path expression.\n\nUse this when you need to access nested data within a JSON structure.\n\nEXAMPLES:\nExtract the email field from the user object in the 'data' JSON column.\n{ \"id\": \"data\", \"path\": '\"user\".\"email\"' }\n\nExtract k8s.cluster.name from the top level of a JSON object.\n{ \"id\": \"data\", \"path\": '\"k8s.cluster.name\"' } \nNotice the quotes around \"k8s.cluster.name\" to handle keys with dots.\n\nExtract service.name from the top level of a JSON object.\n{ \"labels\": \"data\", \"path\": '\"service.name\"' }\nNotice the quotes around \"service.name\" to handle keys with dots.\n    "
                      }
                    ]
                  },
                  "description": "An array of source fields that make up the foreign key.\n\nThese fields in the current dataset link to fields in the target dataset.\nFor composite keys, multiple fields are specified.\n\nEXAMPLES:\n- [\"customer_id\"]: A simple foreign key using a single column\n- [\"tenant_id\", \"user_id\"]: A composite foreign key using two columns\n- [{ \"id\": \"data\", \"path\": \"user.id\" }]: A foreign key using a value from a JSON column\n"
                }
              },
              "required": ["label", "srcFields"],
              "description": "A field that represents a link to another dataset via a foreign key relationship.\n\nUse this to define relationships between datasets, such as linking orders to customers.\n\nEXAMPLES:\nDefines a link named \"Customer\" using the customer_id column as the foreign key.\n{ \"label\": \"Customer\", \"srcFields\": [\"customer_id\"] }\n    "
            },
            {
              "type": "object",
              "properties": {
                "isPrimaryKey": {
                  "type": "boolean",
                  "const": true,
                  "description": "A flag indicating that this field represents a primary key.\n\nThis must be set to true for primary key fields.\n        "
                },
                "srcFields": {
                  "type": "array",
                  "items": {
                    "type": "string",
                    "description": "A simple string identifier for a column in a dataset.\n\nThis is the most basic field type, representing a direct reference to a column by its ID.\n\nEXAMPLES:\n- \"timestamp\": Reference to a timestamp column\n- \"status_code\": Reference to a status code column\n- \"user_id\": Reference to a user ID column\n    "
                  },
                  "description": "An array of column IDs that make up the primary key.\n\nFor composite primary keys, multiple columns are specified.\n\nEXAMPLES:\n- [\"id\"]: A simple primary key using a single column\n- [\"tenant_id\", \"user_id\"]: A composite primary key using two columns\n        "
                }
              },
              "required": ["isPrimaryKey", "srcFields"],
              "description": "A field that represents the primary key of a dataset.\n\nUse this to reference the primary key of a dataset, which uniquely identifies each record.\n\nEXAMPLES:\n{ \"isPrimaryKey\": true, \"srcFields\": [\"id\"] }\nThis defines a primary key using the id column.\n    "
            }
          ]
        },
        "y": {
          "description": "The fields to use for the Y-axis. Can include multiple fields to show multiple series of lines.",
          "type": "array",
          "items": {
            "anyOf": [
              {
                "type": "string",
                "description": "A simple string identifier for a column in a dataset.\n\nThis is the most basic field type, representing a direct reference to a column by its ID.\n\nEXAMPLES:\n- \"timestamp\": Reference to a timestamp column\n- \"status_code\": Reference to a status code column\n- \"user_id\": Reference to a user ID column\n    "
              },
              {
                "type": "object",
                "properties": {
                  "id": {
                    "type": "string",
                    "description": "The ID of the base column containing the JSON data.\n\n        This identifies the column that contains the JSON object you want to extract a value from.\n        "
                  },
                  "path": {
                    "type": "string",
                    "description": "A JSON path expression to extract a specific value from a JSON column.\n\nThe path can be one level deep or multiple levels deep.\nThe each level of the path must be quoted, or surrounded by brackets if it is an array index.\nIf one level deep, use the key name directly. If multiple levels deep, use dot notation to navigate.\n\nIMPORTANT: For each level of the path, it must be enclosed in quotes.\n           Do not use brackets for non numeric array indices.\n           Array indices do not need to be quoted and should use bracket notation.\n\nEXAMPLES:\n- \"name\": Extract the 'name' field from the top level of a JSON object\n- \"k8s.cluster.name\": Extract \"k8s.cluster.name\" from the top level of a JSON object. Quotes are required for keys with dots.\n- \"user\".\"email\": Extract the 'email' field from the 'user' object\n- \"items\"[0].\"id\": Extract the 'id' field from the first item in an 'items' array\n"
                  }
                },
                "required": ["id", "path"],
                "description": "A field that references a value within a JSON column using a path expression.\n\nUse this when you need to access nested data within a JSON structure.\n\nEXAMPLES:\nExtract the email field from the user object in the 'data' JSON column.\n{ \"id\": \"data\", \"path\": '\"user\".\"email\"' }\n\nExtract k8s.cluster.name from the top level of a JSON object.\n{ \"id\": \"data\", \"path\": '\"k8s.cluster.name\"' } \nNotice the quotes around \"k8s.cluster.name\" to handle keys with dots.\n\nExtract service.name from the top level of a JSON object.\n{ \"labels\": \"data\", \"path\": '\"service.name\"' }\nNotice the quotes around \"service.name\" to handle keys with dots.\n    "
              },
              {
                "type": "object",
                "properties": {
                  "label": {
                    "type": "string",
                    "description": "A human-readable label for the link field.\n\nThis is used for display purposes in the UI and helps identify what the link represents.\n\nEXAMPLES:\n- \"Customer\": A link to a customer record\n- \"Parent Order\": A link to a parent order\n        "
                  },
                  "srcFields": {
                    "type": "array",
                    "items": {
                      "anyOf": [
                        {
                          "type": "string",
                          "description": "A simple string identifier for a column in a dataset.\n\nThis is the most basic field type, representing a direct reference to a column by its ID.\n\nEXAMPLES:\n- \"timestamp\": Reference to a timestamp column\n- \"status_code\": Reference to a status code column\n- \"user_id\": Reference to a user ID column\n    "
                        },
                        {
                          "type": "object",
                          "properties": {
                            "id": {
                              "type": "string",
                              "description": "The ID of the base column containing the JSON data.\n\n        This identifies the column that contains the JSON object you want to extract a value from.\n        "
                            },
                            "path": {
                              "type": "string",
                              "description": "A JSON path expression to extract a specific value from a JSON column.\n\nThe path can be one level deep or multiple levels deep.\nThe each level of the path must be quoted, or surrounded by brackets if it is an array index.\nIf one level deep, use the key name directly. If multiple levels deep, use dot notation to navigate.\n\nIMPORTANT: For each level of the path, it must be enclosed in quotes.\n           Do not use brackets for non numeric array indices.\n           Array indices do not need to be quoted and should use bracket notation.\n\nEXAMPLES:\n- \"name\": Extract the 'name' field from the top level of a JSON object\n- \"k8s.cluster.name\": Extract \"k8s.cluster.name\" from the top level of a JSON object. Quotes are required for keys with dots.\n- \"user\".\"email\": Extract the 'email' field from the 'user' object\n- \"items\"[0].\"id\": Extract the 'id' field from the first item in an 'items' array\n"
                            }
                          },
                          "required": ["id", "path"],
                          "description": "A field that references a value within a JSON column using a path expression.\n\nUse this when you need to access nested data within a JSON structure.\n\nEXAMPLES:\nExtract the email field from the user object in the 'data' JSON column.\n{ \"id\": \"data\", \"path\": '\"user\".\"email\"' }\n\nExtract k8s.cluster.name from the top level of a JSON object.\n{ \"id\": \"data\", \"path\": '\"k8s.cluster.name\"' } \nNotice the quotes around \"k8s.cluster.name\" to handle keys with dots.\n\nExtract service.name from the top level of a JSON object.\n{ \"labels\": \"data\", \"path\": '\"service.name\"' }\nNotice the quotes around \"service.name\" to handle keys with dots.\n    "
                        }
                      ]
                    },
                    "description": "An array of source fields that make up the foreign key.\n\nThese fields in the current dataset link to fields in the target dataset.\nFor composite keys, multiple fields are specified.\n\nEXAMPLES:\n- [\"customer_id\"]: A simple foreign key using a single column\n- [\"tenant_id\", \"user_id\"]: A composite foreign key using two columns\n- [{ \"id\": \"data\", \"path\": \"user.id\" }]: A foreign key using a value from a JSON column\n"
                  }
                },
                "required": ["label", "srcFields"],
                "description": "A field that represents a link to another dataset via a foreign key relationship.\n\nUse this to define relationships between datasets, such as linking orders to customers.\n\nEXAMPLES:\nDefines a link named \"Customer\" using the customer_id column as the foreign key.\n{ \"label\": \"Customer\", \"srcFields\": [\"customer_id\"] }\n    "
              },
              {
                "type": "object",
                "properties": {
                  "isPrimaryKey": {
                    "type": "boolean",
                    "const": true,
                    "description": "A flag indicating that this field represents a primary key.\n\nThis must be set to true for primary key fields.\n        "
                  },
                  "srcFields": {
                    "type": "array",
                    "items": {
                      "type": "string",
                      "description": "A simple string identifier for a column in a dataset.\n\nThis is the most basic field type, representing a direct reference to a column by its ID.\n\nEXAMPLES:\n- \"timestamp\": Reference to a timestamp column\n- \"status_code\": Reference to a status code column\n- \"user_id\": Reference to a user ID column\n    "
                    },
                    "description": "An array of column IDs that make up the primary key.\n\nFor composite primary keys, multiple columns are specified.\n\nEXAMPLES:\n- [\"id\"]: A simple primary key using a single column\n- [\"tenant_id\", \"user_id\"]: A composite primary key using two columns\n        "
                  }
                },
                "required": ["isPrimaryKey", "srcFields"],
                "description": "A field that represents the primary key of a dataset.\n\nUse this to reference the primary key of a dataset, which uniquely identifies each record.\n\nEXAMPLES:\n{ \"isPrimaryKey\": true, \"srcFields\": [\"id\"] }\nThis defines a primary key using the id column.\n    "
              }
            ],
            "description": "A union type representing any valid field reference in the system.\n\nThis can be one of the following types:\n1. A simple column ID string\n2. A JSON path reference with base column ID and path \n3. A link field with a label and source fields\n4. A primary key field with a list of source fields\n\n"
          }
        },
        "xConfig": {
          "type": "object",
          "properties": {
            "field": {
              "description": "The field to use for this coordinate axis. Determines which data field's values will be mapped to this axis.",
              "anyOf": [
                {
                  "type": "string",
                  "description": "A simple string identifier for a column in a dataset.\n\nThis is the most basic field type, representing a direct reference to a column by its ID.\n\nEXAMPLES:\n- \"timestamp\": Reference to a timestamp column\n- \"status_code\": Reference to a status code column\n- \"user_id\": Reference to a user ID column\n    "
                },
                {
                  "type": "object",
                  "properties": {
                    "id": {
                      "type": "string",
                      "description": "The ID of the base column containing the JSON data.\n\n        This identifies the column that contains the JSON object you want to extract a value from.\n        "
                    },
                    "path": {
                      "type": "string",
                      "description": "A JSON path expression to extract a specific value from a JSON column.\n\nThe path can be one level deep or multiple levels deep.\nThe each level of the path must be quoted, or surrounded by brackets if it is an array index.\nIf one level deep, use the key name directly. If multiple levels deep, use dot notation to navigate.\n\nIMPORTANT: For each level of the path, it must be enclosed in quotes.\n           Do not use brackets for non numeric array indices.\n           Array indices do not need to be quoted and should use bracket notation.\n\nEXAMPLES:\n- \"name\": Extract the 'name' field from the top level of a JSON object\n- \"k8s.cluster.name\": Extract \"k8s.cluster.name\" from the top level of a JSON object. Quotes are required for keys with dots.\n- \"user\".\"email\": Extract the 'email' field from the 'user' object\n- \"items\"[0].\"id\": Extract the 'id' field from the first item in an 'items' array\n"
                    }
                  },
                  "required": ["id", "path"],
                  "description": "A field that references a value within a JSON column using a path expression.\n\nUse this when you need to access nested data within a JSON structure.\n\nEXAMPLES:\nExtract the email field from the user object in the 'data' JSON column.\n{ \"id\": \"data\", \"path\": '\"user\".\"email\"' }\n\nExtract k8s.cluster.name from the top level of a JSON object.\n{ \"id\": \"data\", \"path\": '\"k8s.cluster.name\"' } \nNotice the quotes around \"k8s.cluster.name\" to handle keys with dots.\n\nExtract service.name from the top level of a JSON object.\n{ \"labels\": \"data\", \"path\": '\"service.name\"' }\nNotice the quotes around \"service.name\" to handle keys with dots.\n    "
                },
                {
                  "type": "object",
                  "properties": {
                    "label": {
                      "type": "string",
                      "description": "A human-readable label for the link field.\n\nThis is used for display purposes in the UI and helps identify what the link represents.\n\nEXAMPLES:\n- \"Customer\": A link to a customer record\n- \"Parent Order\": A link to a parent order\n        "
                    },
                    "srcFields": {
                      "type": "array",
                      "items": {
                        "anyOf": [
                          {
                            "type": "string",
                            "description": "A simple string identifier for a column in a dataset.\n\nThis is the most basic field type, representing a direct reference to a column by its ID.\n\nEXAMPLES:\n- \"timestamp\": Reference to a timestamp column\n- \"status_code\": Reference to a status code column\n- \"user_id\": Reference to a user ID column\n    "
                          },
                          {
                            "type": "object",
                            "properties": {
                              "id": {
                                "type": "string",
                                "description": "The ID of the base column containing the JSON data.\n\n        This identifies the column that contains the JSON object you want to extract a value from.\n        "
                              },
                              "path": {
                                "type": "string",
                                "description": "A JSON path expression to extract a specific value from a JSON column.\n\nThe path can be one level deep or multiple levels deep.\nThe each level of the path must be quoted, or surrounded by brackets if it is an array index.\nIf one level deep, use the key name directly. If multiple levels deep, use dot notation to navigate.\n\nIMPORTANT: For each level of the path, it must be enclosed in quotes.\n           Do not use brackets for non numeric array indices.\n           Array indices do not need to be quoted and should use bracket notation.\n\nEXAMPLES:\n- \"name\": Extract the 'name' field from the top level of a JSON object\n- \"k8s.cluster.name\": Extract \"k8s.cluster.name\" from the top level of a JSON object. Quotes are required for keys with dots.\n- \"user\".\"email\": Extract the 'email' field from the 'user' object\n- \"items\"[0].\"id\": Extract the 'id' field from the first item in an 'items' array\n"
                              }
                            },
                            "required": ["id", "path"],
                            "description": "A field that references a value within a JSON column using a path expression.\n\nUse this when you need to access nested data within a JSON structure.\n\nEXAMPLES:\nExtract the email field from the user object in the 'data' JSON column.\n{ \"id\": \"data\", \"path\": '\"user\".\"email\"' }\n\nExtract k8s.cluster.name from the top level of a JSON object.\n{ \"id\": \"data\", \"path\": '\"k8s.cluster.name\"' } \nNotice the quotes around \"k8s.cluster.name\" to handle keys with dots.\n\nExtract service.name from the top level of a JSON object.\n{ \"labels\": \"data\", \"path\": '\"service.name\"' }\nNotice the quotes around \"service.name\" to handle keys with dots.\n    "
                          }
                        ]
                      },
                      "description": "An array of source fields that make up the foreign key.\n\nThese fields in the current dataset link to fields in the target dataset.\nFor composite keys, multiple fields are specified.\n\nEXAMPLES:\n- [\"customer_id\"]: A simple foreign key using a single column\n- [\"tenant_id\", \"user_id\"]: A composite foreign key using two columns\n- [{ \"id\": \"data\", \"path\": \"user.id\" }]: A foreign key using a value from a JSON column\n"
                    }
                  },
                  "required": ["label", "srcFields"],
                  "description": "A field that represents a link to another dataset via a foreign key relationship.\n\nUse this to define relationships between datasets, such as linking orders to customers.\n\nEXAMPLES:\nDefines a link named \"Customer\" using the customer_id column as the foreign key.\n{ \"label\": \"Customer\", \"srcFields\": [\"customer_id\"] }\n    "
                },
                {
                  "type": "object",
                  "properties": {
                    "isPrimaryKey": {
                      "type": "boolean",
                      "const": true,
                      "description": "A flag indicating that this field represents a primary key.\n\nThis must be set to true for primary key fields.\n        "
                    },
                    "srcFields": {
                      "type": "array",
                      "items": {
                        "type": "string",
                        "description": "A simple string identifier for a column in a dataset.\n\nThis is the most basic field type, representing a direct reference to a column by its ID.\n\nEXAMPLES:\n- \"timestamp\": Reference to a timestamp column\n- \"status_code\": Reference to a status code column\n- \"user_id\": Reference to a user ID column\n    "
                      },
                      "description": "An array of column IDs that make up the primary key.\n\nFor composite primary keys, multiple columns are specified.\n\nEXAMPLES:\n- [\"id\"]: A simple primary key using a single column\n- [\"tenant_id\", \"user_id\"]: A composite primary key using two columns\n        "
                    }
                  },
                  "required": ["isPrimaryKey", "srcFields"],
                  "description": "A field that represents the primary key of a dataset.\n\nUse this to reference the primary key of a dataset, which uniquely identifies each record.\n\nEXAMPLES:\n{ \"isPrimaryKey\": true, \"srcFields\": [\"id\"] }\nThis defines a primary key using the id column.\n    "
                }
              ]
            },
            "label": {
              "type": "string",
              "description": "The display label for the axis. Use the field name used in the visualization binding \nfor the axis or a more descriptive short name that represents the values being plotted. Important: Do not\ninclude unit information."
            },
            "valueFormat": {
              "description": "The formatting options for the axis values. Controls how values are displayed on the axis and in tooltips.",
              "type": "object",
              "properties": {
                "options": {
                  "type": "object",
                  "properties": {
                    "options": {
                      "type": "object",
                      "properties": {
                        "style": {
                          "description": "The style of the number format. Determines the general form of the output. If metricUnit is present, this must be set to 'unit'",
                          "anyOf": [
                            {
                              "type": "string",
                              "const": "currency"
                            },
                            {
                              "type": "string",
                              "const": "unit"
                            },
                            {
                              "type": "string",
                              "const": "percent"
                            },
                            {
                              "type": "string",
                              "const": "decimal"
                            }
                          ]
                        },
                        "metricUnit": {
                          "description": "The metric unit (e.g. ns, bytes, etc), only provided if the input is a metric dataset \nand the relevant column has a metric unit. Otherwise this shouldn't be present.",
                          "type": "string"
                        },
                        "metricUnitPrecision": {
                          "description": "The precision to use for the metric unit. Only valid if metricUnit is present.",
                          "type": "number"
                        },
                        "currency": {
                          "description": "The currency to use for the format.",
                          "type": "string"
                        },
                        "currencyDisplay": {
                          "description": "The display style for the currency.",
                          "anyOf": [
                            {
                              "type": "string",
                              "const": "code"
                            },
                            {
                              "type": "string",
                              "const": "symbol"
                            },
                            {
                              "type": "string",
                              "const": "narrowSymbol"
                            },
                            {
                              "type": "string",
                              "const": "name"
                            }
                          ]
                        }
                      }
                    },
                    "locale": {
                      "description": "The i18n locale to use for formatting. If not provided, the browser locale is used.",
                      "type": "string"
                    }
                  },
                  "required": ["options"],
                  "description": "The formatting options for the value. Controls how the value is displayed."
                }
              },
              "required": ["options"]
            },
            "range": {
              "description": "The value range for the axis. Controls the domain of the axis scale.",
              "type": "object",
              "properties": {
                "min": {
                  "description": "The minimum value for the axis range. Sets the lower bound of the axis.",
                  "type": "number"
                },
                "max": {
                  "description": "The maximum value for the axis range. Sets the upper bound of the axis.",
                  "type": "number"
                }
              }
            },
            "scaleType": {
              "description": "The scale configuration for the axis. Controls how data values are mapped to positions on the axis.",
              "type": "object",
              "properties": {
                "type": {
                  "anyOf": [
                    {
                      "type": "string",
                      "const": "log",
                      "description": "Uses a logarithmic scale. Good for data that spans multiple orders of magnitude."
                    },
                    {
                      "type": "string",
                      "const": "linear",
                      "description": "Uses a linear scale. The standard scale type for most data."
                    }
                  ],
                  "description": "The type of scale to use for this axis. Determines how data values are mapped to positions on the axis."
                },
                "base": {
                  "description": "The base for logarithmic scales. Only used when type is 'log'. Default is 10.",
                  "type": "number"
                }
              },
              "required": ["type"]
            },
            "show": {
              "default": true,
              "description": "Whether to show the axis. Set to false to hide the axis completely.",
              "type": "boolean"
            }
          },
          "required": ["label"],
          "description": "Configuration for the X-axis. Controls how the X-axis is displayed and scaled."
        },
        "yConfig": {
          "type": "object",
          "properties": {
            "field": {
              "description": "The field to use for this coordinate axis. Determines which data field's values will be mapped to this axis.",
              "anyOf": [
                {
                  "type": "string",
                  "description": "A simple string identifier for a column in a dataset.\n\nThis is the most basic field type, representing a direct reference to a column by its ID.\n\nEXAMPLES:\n- \"timestamp\": Reference to a timestamp column\n- \"status_code\": Reference to a status code column\n- \"user_id\": Reference to a user ID column\n    "
                },
                {
                  "type": "object",
                  "properties": {
                    "id": {
                      "type": "string",
                      "description": "The ID of the base column containing the JSON data.\n\n        This identifies the column that contains the JSON object you want to extract a value from.\n        "
                    },
                    "path": {
                      "type": "string",
                      "description": "A JSON path expression to extract a specific value from a JSON column.\n\nThe path can be one level deep or multiple levels deep.\nThe each level of the path must be quoted, or surrounded by brackets if it is an array index.\nIf one level deep, use the key name directly. If multiple levels deep, use dot notation to navigate.\n\nIMPORTANT: For each level of the path, it must be enclosed in quotes.\n           Do not use brackets for non numeric array indices.\n           Array indices do not need to be quoted and should use bracket notation.\n\nEXAMPLES:\n- \"name\": Extract the 'name' field from the top level of a JSON object\n- \"k8s.cluster.name\": Extract \"k8s.cluster.name\" from the top level of a JSON object. Quotes are required for keys with dots.\n- \"user\".\"email\": Extract the 'email' field from the 'user' object\n- \"items\"[0].\"id\": Extract the 'id' field from the first item in an 'items' array\n"
                    }
                  },
                  "required": ["id", "path"],
                  "description": "A field that references a value within a JSON column using a path expression.\n\nUse this when you need to access nested data within a JSON structure.\n\nEXAMPLES:\nExtract the email field from the user object in the 'data' JSON column.\n{ \"id\": \"data\", \"path\": '\"user\".\"email\"' }\n\nExtract k8s.cluster.name from the top level of a JSON object.\n{ \"id\": \"data\", \"path\": '\"k8s.cluster.name\"' } \nNotice the quotes around \"k8s.cluster.name\" to handle keys with dots.\n\nExtract service.name from the top level of a JSON object.\n{ \"labels\": \"data\", \"path\": '\"service.name\"' }\nNotice the quotes around \"service.name\" to handle keys with dots.\n    "
                },
                {
                  "type": "object",
                  "properties": {
                    "label": {
                      "type": "string",
                      "description": "A human-readable label for the link field.\n\nThis is used for display purposes in the UI and helps identify what the link represents.\n\nEXAMPLES:\n- \"Customer\": A link to a customer record\n- \"Parent Order\": A link to a parent order\n        "
                    },
                    "srcFields": {
                      "type": "array",
                      "items": {
                        "anyOf": [
                          {
                            "type": "string",
                            "description": "A simple string identifier for a column in a dataset.\n\nThis is the most basic field type, representing a direct reference to a column by its ID.\n\nEXAMPLES:\n- \"timestamp\": Reference to a timestamp column\n- \"status_code\": Reference to a status code column\n- \"user_id\": Reference to a user ID column\n    "
                          },
                          {
                            "type": "object",
                            "properties": {
                              "id": {
                                "type": "string",
                                "description": "The ID of the base column containing the JSON data.\n\n        This identifies the column that contains the JSON object you want to extract a value from.\n        "
                              },
                              "path": {
                                "type": "string",
                                "description": "A JSON path expression to extract a specific value from a JSON column.\n\nThe path can be one level deep or multiple levels deep.\nThe each level of the path must be quoted, or surrounded by brackets if it is an array index.\nIf one level deep, use the key name directly. If multiple levels deep, use dot notation to navigate.\n\nIMPORTANT: For each level of the path, it must be enclosed in quotes.\n           Do not use brackets for non numeric array indices.\n           Array indices do not need to be quoted and should use bracket notation.\n\nEXAMPLES:\n- \"name\": Extract the 'name' field from the top level of a JSON object\n- \"k8s.cluster.name\": Extract \"k8s.cluster.name\" from the top level of a JSON object. Quotes are required for keys with dots.\n- \"user\".\"email\": Extract the 'email' field from the 'user' object\n- \"items\"[0].\"id\": Extract the 'id' field from the first item in an 'items' array\n"
                              }
                            },
                            "required": ["id", "path"],
                            "description": "A field that references a value within a JSON column using a path expression.\n\nUse this when you need to access nested data within a JSON structure.\n\nEXAMPLES:\nExtract the email field from the user object in the 'data' JSON column.\n{ \"id\": \"data\", \"path\": '\"user\".\"email\"' }\n\nExtract k8s.cluster.name from the top level of a JSON object.\n{ \"id\": \"data\", \"path\": '\"k8s.cluster.name\"' } \nNotice the quotes around \"k8s.cluster.name\" to handle keys with dots.\n\nExtract service.name from the top level of a JSON object.\n{ \"labels\": \"data\", \"path\": '\"service.name\"' }\nNotice the quotes around \"service.name\" to handle keys with dots.\n    "
                          }
                        ]
                      },
                      "description": "An array of source fields that make up the foreign key.\n\nThese fields in the current dataset link to fields in the target dataset.\nFor composite keys, multiple fields are specified.\n\nEXAMPLES:\n- [\"customer_id\"]: A simple foreign key using a single column\n- [\"tenant_id\", \"user_id\"]: A composite foreign key using two columns\n- [{ \"id\": \"data\", \"path\": \"user.id\" }]: A foreign key using a value from a JSON column\n"
                    }
                  },
                  "required": ["label", "srcFields"],
                  "description": "A field that represents a link to another dataset via a foreign key relationship.\n\nUse this to define relationships between datasets, such as linking orders to customers.\n\nEXAMPLES:\nDefines a link named \"Customer\" using the customer_id column as the foreign key.\n{ \"label\": \"Customer\", \"srcFields\": [\"customer_id\"] }\n    "
                },
                {
                  "type": "object",
                  "properties": {
                    "isPrimaryKey": {
                      "type": "boolean",
                      "const": true,
                      "description": "A flag indicating that this field represents a primary key.\n\nThis must be set to true for primary key fields.\n        "
                    },
                    "srcFields": {
                      "type": "array",
                      "items": {
                        "type": "string",
                        "description": "A simple string identifier for a column in a dataset.\n\nThis is the most basic field type, representing a direct reference to a column by its ID.\n\nEXAMPLES:\n- \"timestamp\": Reference to a timestamp column\n- \"status_code\": Reference to a status code column\n- \"user_id\": Reference to a user ID column\n    "
                      },
                      "description": "An array of column IDs that make up the primary key.\n\nFor composite primary keys, multiple columns are specified.\n\nEXAMPLES:\n- [\"id\"]: A simple primary key using a single column\n- [\"tenant_id\", \"user_id\"]: A composite primary key using two columns\n        "
                    }
                  },
                  "required": ["isPrimaryKey", "srcFields"],
                  "description": "A field that represents the primary key of a dataset.\n\nUse this to reference the primary key of a dataset, which uniquely identifies each record.\n\nEXAMPLES:\n{ \"isPrimaryKey\": true, \"srcFields\": [\"id\"] }\nThis defines a primary key using the id column.\n    "
                }
              ]
            },
            "label": {
              "type": "string",
              "description": "The display label for the axis. Use the field name used in the visualization binding \nfor the axis or a more descriptive short name that represents the values being plotted. Important: Do not\ninclude unit information."
            },
            "valueFormat": {
              "description": "The formatting options for the axis values. Controls how values are displayed on the axis and in tooltips.",
              "type": "object",
              "properties": {
                "options": {
                  "type": "object",
                  "properties": {
                    "options": {
                      "type": "object",
                      "properties": {
                        "style": {
                          "description": "The style of the number format. Determines the general form of the output. If metricUnit is present, this must be set to 'unit'",
                          "anyOf": [
                            {
                              "type": "string",
                              "const": "currency"
                            },
                            {
                              "type": "string",
                              "const": "unit"
                            },
                            {
                              "type": "string",
                              "const": "percent"
                            },
                            {
                              "type": "string",
                              "const": "decimal"
                            }
                          ]
                        },
                        "metricUnit": {
                          "description": "The metric unit (e.g. ns, bytes, etc), only provided if the input is a metric dataset \nand the relevant column has a metric unit. Otherwise this shouldn't be present.",
                          "type": "string"
                        },
                        "metricUnitPrecision": {
                          "description": "The precision to use for the metric unit. Only valid if metricUnit is present.",
                          "type": "number"
                        },
                        "currency": {
                          "description": "The currency to use for the format.",
                          "type": "string"
                        },
                        "currencyDisplay": {
                          "description": "The display style for the currency.",
                          "anyOf": [
                            {
                              "type": "string",
                              "const": "code"
                            },
                            {
                              "type": "string",
                              "const": "symbol"
                            },
                            {
                              "type": "string",
                              "const": "narrowSymbol"
                            },
                            {
                              "type": "string",
                              "const": "name"
                            }
                          ]
                        }
                      }
                    },
                    "locale": {
                      "description": "The i18n locale to use for formatting. If not provided, the browser locale is used.",
                      "type": "string"
                    }
                  },
                  "required": ["options"],
                  "description": "The formatting options for the value. Controls how the value is displayed."
                }
              },
              "required": ["options"]
            },
            "range": {
              "description": "The value range for the axis. Controls the domain of the axis scale.",
              "type": "object",
              "properties": {
                "min": {
                  "description": "The minimum value for the axis range. Sets the lower bound of the axis.",
                  "type": "number"
                },
                "max": {
                  "description": "The maximum value for the axis range. Sets the upper bound of the axis.",
                  "type": "number"
                }
              }
            },
            "scaleType": {
              "description": "The scale configuration for the axis. Controls how data values are mapped to positions on the axis.",
              "type": "object",
              "properties": {
                "type": {
                  "anyOf": [
                    {
                      "type": "string",
                      "const": "log",
                      "description": "Uses a logarithmic scale. Good for data that spans multiple orders of magnitude."
                    },
                    {
                      "type": "string",
                      "const": "linear",
                      "description": "Uses a linear scale. The standard scale type for most data."
                    }
                  ],
                  "description": "The type of scale to use for this axis. Determines how data values are mapped to positions on the axis."
                },
                "base": {
                  "description": "The base for logarithmic scales. Only used when type is 'log'. Default is 10.",
                  "type": "number"
                }
              },
              "required": ["type"]
            },
            "show": {
              "default": true,
              "description": "Whether to show the axis. Set to false to hide the axis completely.",
              "type": "boolean"
            }
          },
          "required": ["label"],
          "description": "Configuration for the Y-axis. Controls how the Y-axis is displayed and scaled."
        },
        "lineConfig": {
          "description": "Omit unless explicitly needed. Configuration for line appearance. Controls how lines are displayed in the chart.",
          "type": "object",
          "properties": {
            "showPoints": {
              "description": "Whether to show points on the lines. Set to true to display markers at each data point.",
              "type": "boolean"
            },
            "showValues": {
              "description": "Whether to show values next to points. Set to true to display data values alongside each point.",
              "type": "boolean"
            },
            "curve": {
              "description": "The type of curve to use for lines. Controls how points are connected in the line chart.",
              "anyOf": [
                {
                  "type": "string",
                  "const": "Linear",
                  "description": "Connects data points with straight lines. The simplest curve type that creates sharp angles between segments."
                },
                {
                  "type": "string",
                  "const": "Curve",
                  "description": "Creates a curve that preserves monotonicity. Ensures that if data is increasing/decreasing, the curve will not create artificial peaks or valleys."
                },
                {
                  "type": "string",
                  "const": "Step",
                  "description": "Creates a step function curve with steps at the midpoints. Data changes appear as horizontal and vertical lines."
                }
              ]
            }
          }
        },
        "horizontal": {
          "description": "Whether to swap the X and Y axes. Set to true to create a horizontal line chart.",
          "type": "boolean"
        },
        "legend": {
          "description": "Configuration for the chart legend. Controls the appearance and behavior of the legend.",
          "type": "object",
          "properties": {
            "visible": {
              "description": "Whether to show the legend at all. Set to false to hide the legend completely.Time series charts with more than 1 series should have this true. If X or Y fields are the same as the group key, then this should be false.",
              "anyOf": [
                {
                  "type": "boolean"
                },
                {
                  "type": "null"
                }
              ]
            },
            "placement": {
              "description": "The position of the legend relative to the visualization. Controls where the legend appears.",
              "anyOf": [
                {
                  "anyOf": [
                    {
                      "type": "string",
                      "const": "top",
                      "description": "Places the legend at the top of the visualization. Good for horizontal legends with many items."
                    },
                    {
                      "type": "string",
                      "const": "right",
                      "description": "Places the legend at the right of the visualization. The default and most common position."
                    },
                    {
                      "type": "string",
                      "const": "bottom",
                      "description": "Places the legend at the bottom of the visualization. Good for horizontal legends with many items."
                    },
                    {
                      "type": "string",
                      "const": "left",
                      "description": "Places the legend at the left of the visualization. Useful when right side has other elements."
                    }
                  ]
                },
                {
                  "type": "null"
                }
              ]
            }
          }
        }
      },
      "required": ["xConfig", "yConfig"]
    }
  },
  "required": ["lineChart"]
}
```
