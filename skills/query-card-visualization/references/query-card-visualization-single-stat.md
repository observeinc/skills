# singleStat

JSON schema for the `singleStat` visualizationTemplate object. The `visualizationTemplate` value MUST be a single object matching this schema (top-level key `singleStat`). Do not write it from memory; emit exactly this shape.

```json
{
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "properties": {
        "singleStat": {
            "description": "Configuration for single stat visualizations in Vega. If the user wants to display a single number/result, use this visualization.",
            "type": "object",
            "properties": {
                "y": {
                    "description": "The field to use for the value. Determines which data field's value will be displayed.",
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
                "yConfig": {
                    "description": "Configuration for the value. Controls how the value is displayed and formatted.",
                    "type": "object",
                    "properties": {
                        "format": {
                            "description": "The formatting options for value.",
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
                        }
                    }
                },
                "aggregation": {
                    "description": "The aggregation function to use to get the value if there are multiple data points (e.g. time-series). Defaults to 'last-non-null' if not provided.",
                    "type": "string",
                    "enum": ["last-non-null", "last", "first-non-null", "first"]
                }
            }
        }
    },
    "required": ["singleStat"]
}
```
