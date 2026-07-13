# changeOverTime

JSON schema for the `changeOverTime` visualizationTemplate object. The `visualizationTemplate` value MUST be a single object matching this schema (top-level key `changeOverTime`). Do not write it from memory; emit exactly this shape.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "changeOverTime": {
      "description": "Configuration for change over time visualizations in Vega. Displays how values change between two points in time.",
      "type": "object",
      "properties": {
        "x": {
          "description": "The field to use for the time axis. Determines which data field's values will be used as time points.",
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
        "key": {
          "description": "The field to use for item keys. Determines which data field's values will be used as item identifiers.",
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
        "value": {
          "description": "The field to use for item values. Determines which data field's values will be tracked over time.",
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
        "keyConfig": {
          "description": "Configuration for the key column. Controls how keys are displayed and sorted.",
          "type": "object",
          "properties": {
            "label": {
              "description": "The display label for the key column. Used as a header for the key column.",
              "type": "string"
            },
            "order": {
              "description": "The sort order for items. Controls whether items are sorted in ascending or descending order.",
              "anyOf": [
                {
                  "type": "string",
                  "const": "descending",
                  "description": "Sorts items in descending order. Shows highest changes first."
                },
                {
                  "type": "string",
                  "const": "ascending",
                  "description": "Sorts items in ascending order. Shows lowest changes first."
                }
              ]
            }
          }
        },
        "valueConfig": {
          "description": "Configuration for the value column. Controls how values are displayed and formatted.",
          "type": "object",
          "properties": {
            "range": {
              "description": "The value range for display. Controls the domain of value visualization.",
              "type": "object",
              "properties": {
                "min": {
                  "description": "The minimum value for the value range. Sets the lower bound of value display.",
                  "type": "number"
                },
                "max": {
                  "description": "The maximum value for the value range. Sets the upper bound of value display.",
                  "type": "number"
                }
              }
            },
            "scaleType": {
              "description": "The scale configuration for values. Controls how values are mapped to visual properties.",
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
                  "description": "The type of scale to use for values. Determines how values are mapped to visual properties."
                },
                "base": {
                  "description": "The base for logarithmic scales. Only used when type is 'log'. Default is 10.",
                  "type": "number"
                }
              },
              "required": ["type"]
            },
            "format": {
              "description": "The formatting options for values. Controls how values are displayed in the visualization."
            },
            "showAxis": {
              "description": "Whether to show an axis for values. Set to true to display a scale for the values.",
              "type": "boolean"
            },
            "label": {
              "description": "The display label for the value column. Used as a header for the value column.",
              "type": "string"
            }
          }
        },
        "barConfig": {
          "description": "Configuration for bar appearance. Controls how bars are displayed in the visualization.",
          "type": "object",
          "properties": {
            "bandSize": {
              "description": "The width of bars as a proportion of the available space. Controls the thickness of bars relative to the space between them.",
              "type": "number"
            }
          }
        }
      }
    }
  },
  "required": ["changeOverTime"]
}
```
