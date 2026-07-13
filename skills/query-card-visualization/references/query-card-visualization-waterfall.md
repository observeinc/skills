# waterfall

JSON schema for the `waterfall` visualizationTemplate object. The `visualizationTemplate` value MUST be a single object matching this schema (top-level key `waterfall`). Do not write it from memory; emit exactly this shape.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "waterfall": {
      "description": "Configuration for waterfall visualizations. Displays hierarchical span/timing data with parent-child relationships in a waterfall chart format. Ideal for distributed traces, job hierarchies, or any data with nested operations and time ranges. Each row shows a span with its name, optional metadata columns, and a horizontal bar indicating its timing relative to the overall time range.",
      "type": "object",
      "properties": {
        "source": {
          "description": "The data source configuration for the waterfall chart.",
          "type": "object",
          "properties": {
            "table": {
              "description": "The table binding configuration for the waterfall chart. Defines the time and hierarchy fields.",
              "type": "object",
              "properties": {
                "validFromField": {
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
                  "description": "The field containing the start time of each span/node. This is required and determines when each item begins on the time axis."
                },
                "validToField": {
                  "description": "The field containing the end time of each span/node. Determines when each item ends on the time axis. If not provided, duration may be calculated differently.",
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
                "idField": {
                  "description": "The field containing the unique identifier for each span/node. Used to establish parent-child relationships and for selection.",
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
                "parentIdField": {
                  "description": "The field containing the parent span/node ID. Used to build the hierarchy and determine nesting depth in the waterfall.",
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
                }
              },
              "required": ["validFromField"]
            }
          },
          "required": ["table"]
        },
        "config": {
          "description": "The configuration for the waterfall visualization appearance and behavior.",
          "type": "object",
          "properties": {
            "labelField": {
              "description": "The field to use as the display label for each span/node in the waterfall. This appears in the name column.",
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
            "colorField": {
              "description": "The field to use for color-coding spans/nodes. Different values will receive different colors for visual grouping.",
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
            "additionalTableFields": {
              "description": "Additional fields to display as columns in the waterfall table. These appear between the name and the timing chart.",
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
            "additionalMetadataFields": {
              "description": "Additional fields to include in the metadata/details view when a span is selected.",
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
            "zebraStriping": {
              "description": "Whether to apply alternating row colors for better readability. Defaults to true.",
              "type": "boolean"
            },
            "showSearchToolbarItem": {
              "description": "Whether to show the search toolbar item for filtering spans. Defaults to true.",
              "type": "boolean"
            }
          }
        }
      },
      "required": ["source", "config"]
    }
  },
  "required": ["waterfall"]
}
```
