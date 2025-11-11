              "description": "Optional. Custom authentication config, including the path to your Python auth logic and\nthe OpenAPI security definitions it uses.\n"
            },
            "base_image": {
              "anyOf": [
                {
                  "type": "string"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Base image to use for the LangGraph API server.\n\nDefaults to langchain/langgraph-api or langchain/langgraphjs-api.\n"
            },
            "checkpointer": {
              "anyOf": [
                {
                  "$ref": "#/$defs/CheckpointerConfig"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Configuration for the built-in checkpointer, which handles checkpointing of state.\n\nIf omitted, no checkpointer is set up (the object store will still be present, however).\n"
            },
            "dependencies": {
              "type": "array",
              "items": {
                "type": "string"
              },
              "description": "List of Python dependencies to install, either from PyPI or local paths.\n"
            },
            "dockerfile_lines": {
              "type": "array",
              "items": {
                "type": "string"
              },
              "description": "Optional. Additional Docker instructions that will be appended to your base Dockerfile.\n\nUseful for installing OS packages, setting environment variables, etc."
            },
            "env": {
              "anyOf": [
                {
                  "type": "object",
                  "additionalProperties": {
                    "type": "string"
                  }
                },
                {
                  "type": "string"
                }
              ],
              "description": "Optional. Environment variables to set for your deployment.\n\n- If given as a dict, keys are variable names and values are their values.\n- If given as a string, it must be a path to a file containing lines in KEY=VALUE format.\n\nenv=\".env\n"
            },
            "graphs": {
              "type": "object",
              "additionalProperties": {
                "type": "string"
              },
              "description": "Optional. Named definitions of graphs, each pointing to a Python object.\n\n\nGraphs can be StateGraph, @entrypoint, or any other Pregel object OR they can point to (async) context\nmanagers that accept a single configuration argument (of type RunnableConfig) and return a pregel object\n(instance of Stategraph, etc.).\n"
            },
            "http": {
              "anyOf": [
                {
                  "$ref": "#/$defs/HttpConfig"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Configuration for the built-in HTTP server, controlling which custom routes are exposed\nand how cross-origin requests are handled.\n"
            },
            "image_distro": {
              "anyOf": [
                {
                  "type": "string",
                  "enum": [
                    "debian",
                    "wolfi"
                  ]
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Linux distribution for the base image.\n\nMust be one of 'wolfi', 'debian', 'bullseye', or 'bookworm'.\nIf omitted, defaults to 'debian' ('latest').\n"
            },
            "keep_pkg_tools": {
              "anyOf": [
                {
                  "type": "boolean"
                },
                {
                  "type": "array",
                  "items": {
                    "type": "string"
                  }
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Control whether to retain Python packaging tools in the final image.\n\nYou can also set to true to include all packaging tools.\n"
            },
            "pip_installer": {
              "anyOf": [
                {
                  "type": "string"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Python package installer to use ('auto', 'pip', 'uv').\n\n"
            },
            "store": {
              "anyOf": [
                {
                  "$ref": "#/$defs/StoreConfig"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Configuration for the built-in long-term memory store, including semantic search indexing.\n\nIf omitted, no vector index is set up (the object store will still be present, however).\n"
            },
            "ui": {
              "anyOf": [
                {
                  "type": "object",
                  "additionalProperties": {
                    "type": "string"
                  }
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Named definitions of UI components emitted by the agent, each pointing to a JS/TS file.\n"
            }
          },
          "required": [
            "node_version",
            "graphs"
          ]
        }
      ]
    },
    "AuthConfig": {
      "title": "AuthConfig",
      "description": "Configuration for custom authentication logic and how it integrates into the OpenAPI spec.",
      "type": "object",
      "properties": {
        "disable_studio_auth": {
          "type": "boolean",
          "description": "Optional. Whether to disable LangSmith API-key authentication for requests originating the Studio.\n\nDefaults to False, meaning that if a particular header is set, the server will verify the `x-api-key` header\nvalue is a valid API key for the deployment's workspace. If `True`, all requests will go through your custom\nauthentication logic, regardless of origin of the request.\n"
        },
        "openapi": {
          "$ref": "#/$defs/SecurityConfig",
          "description": "The security configuration to include in your server's OpenAPI spec.\n\n{\n}\n}\n}\n},\n]\n}\n"
        },
        "path": {
          "type": "string",
          "description": "Required. Path to an instance of the Auth() class that implements custom authentication.\n"
        }
      },
      "required": []
    },
    "SecurityConfig": {
      "title": "SecurityConfig",
      "description": "Configuration for OpenAPI security definitions and requirements.\n\nUseful for specifying global or path-level authentication and authorization flows\n(e.g., OAuth2, API key headers, etc.).",
      "type": "object",
      "properties": {
        "paths": {
          "type": "object",
          "additionalProperties": {
            "type": "object",
            "additionalProperties": {
              "type": "array",
              "items": {
                "type": "object",
                "additionalProperties": {
                  "type": "array",
                  "items": {
                    "type": "string"
                  }
                }
              }
            }
          },
          "description": "Path-specific security overrides.\n\n- Keys that are HTTP methods (e.g., \"GET\", \"POST\"),\n- Values are lists of security definitions (just like `security`) for that method.\n"
        },
        "security": {
          "type": "array",
          "items": {
            "type": "object",
            "additionalProperties": {
              "type": "array",
              "items": {
                "type": "string"
              }
            }
          },
          "description": "Global security requirements across all endpoints.\n\nEach element in the list maps a security scheme (e.g. \"OAuth2\") to a list of scopes (e.g. [\"read\", \"write\"])."
        },
        "securitySchemes": {
          "type": "object",
          "additionalProperties": {
            "type": "object"
          },
          "description": "Describe each security scheme recognized by your OpenAPI spec.\n\nKeys are scheme names (e.g. \"OAuth2\", \"ApiKeyAuth\") and values are their definitions."
        }
      },
      "required": []
    },
    "CheckpointerConfig": {
      "title": "CheckpointerConfig",
      "description": "Configuration for the built-in checkpointer, which handles checkpointing of state.\n\nIf omitted, no checkpointer is set up (the object store will still be present, however).",
      "type": "object",
      "properties": {
        "serde": {
          "anyOf": [
            {
              "$ref": "#/$defs/SerdeConfig"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines the serde configuration.\n\nIf provided, the checkpointer will apply serde settings according to the configuration.\nIf omitted, no serde behavior is configured.\n\nThis configuration requires server version 0.5 or later to take effect.\n"
        },
        "ttl": {
          "anyOf": [
            {
              "$ref": "#/$defs/ThreadTTLConfig"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines the TTL (time-to-live) behavior configuration.\n\nIf provided, the checkpointer will apply TTL settings according to the configuration.\nIf omitted, no TTL behavior is configured.\n"
        }
      },
      "required": []
    },
    "SerdeConfig": {
      "title": "SerdeConfig",
      "description": "Configuration for the built-in serde, which handles checkpointing of state.\n\nIf omitted, no serde is set up (the object store will still be present, however).",
      "type": "object",
      "properties": {
        "allowed_json_modules": {
          "anyOf": [
            {
              "type": "array",
              "items": {
                "type": "array",
                "items": {
                  "type": "string"
                }
              }
            },
            {
              "type": "boolean"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. List of allowed python modules to de-serialize custom objects from.\n\nIf provided, only the specified modules will be allowed to be deserialized.\nIf omitted, no modules are allowed, and the object returned will simply be a json object OR\na deserialized langchain object.\n"
        },
        "pickle_fallback": {
          "type": "boolean",
          "description": "Optional. Whether to allow pickling as a fallback for deserialization.\n\nIf True, pickling will be allowed as a fallback for deserialization.\nIf False, pickling will not be allowed as a fallback for deserialization.\nDefaults to True if not configured."
        }
      },
      "required": []
    },
    "ThreadTTLConfig": {
      "title": "ThreadTTLConfig",
      "description": "Configure a default TTL for checkpointed data within threads.",
      "type": "object",
      "properties": {
        "default_ttl": {
          "anyOf": [
            {
              "type": "number"
            },
            {
              "type": "null"
            }
          ],
          "description": "Default TTL (time-to-live) in minutes for checkpointed data."
        },
        "strategy": {
          "enum": [
            "delete"
          ],
          "description": "Strategy to use for deleting checkpointed data.\n"
        },
        "sweep_interval_minutes": {
          "anyOf": [
            {
              "type": "integer"
            },
            {
              "type": "null"
            }
          ],
          "description": "Interval in minutes between sweep iterations.\nIf omitted, a default interval will be used (typically ~ 5 minutes)."
        }
      },
      "required": []
    },
    "HttpConfig": {
      "title": "HttpConfig",
      "description": "Configuration for the built-in HTTP server that powers your deployment's routes and endpoints.",
      "type": "object",
      "properties": {
        "app": {
          "type": "string",
          "description": "Optional. Import path to a custom Starlette/FastAPI application to mount.\n"
        },
        "configurable_headers": {
          "anyOf": [
            {
              "$ref": "#/$defs/ConfigurableHeaderConfig"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines how headers are treated for a run's configuration.\n\nYou can include or exclude headers as configurable values to condition your\nagent's behavior or permissions on a request's headers."
        },
        "cors": {
          "anyOf": [
            {
              "$ref": "#/$defs/CorsConfig"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines CORS restrictions. If omitted, no special rules are set and\ncross-origin behavior depends on default server settings.\n"
        },
        "disable_assistants": {
          "type": "boolean",
          "description": "Optional. If `True`, /assistants routes are removed from the server.\n\nDefault is False (meaning /assistants is enabled).\n"
        },
        "disable_mcp": {
          "type": "boolean",
          "description": "Optional. If `True`, /mcp routes are removed, disabling the MCP server.\n\nDefault is False.\n"
        },
        "disable_meta": {
          "type": "boolean",
          "description": "Optional. Remove meta endpoints.\n\n\nDefault is False.\n"
        },
        "disable_runs": {
          "type": "boolean",
          "description": "Optional. If `True`, /runs routes are removed.\n\nDefault is False.\n"
        },
        "disable_store": {
          "type": "boolean",
          "description": "Optional. If `True`, /store routes are removed, disabling direct store interactions via HTTP.\n\nDefault is False.\n"
        },
        "disable_threads": {
          "type": "boolean",
          "description": "Optional. If `True`, /threads routes are removed.\n\nDefault is False.\n"
        },
        "enable_custom_route_auth": {
          "type": "boolean",
          "description": "Optional. If `True`, authentication is enabled for custom routes,\nnot just the routes that are protected by default.\n(Routes protected by default include /assistants, /threads, and /runs).\n\nDefault is False. This flag only affects authentication behavior\nif `app` is provided and contains custom routes.\n"
        },
        "logging_headers": {
          "anyOf": [
            {
              "$ref": "#/$defs/ConfigurableHeaderConfig"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines which headers are excluded from logging."
        },
        "middleware_order": {
          "anyOf": [
            {
              "enum": [
                "auth_first",
                "middleware_first"
              ]
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines the order in which to apply server customizations.\n"
        }
      },
      "required": []
    },
    "ConfigurableHeaderConfig": {
      "title": "ConfigurableHeaderConfig",
      "description": "Customize which headers to include as configurable values in your runs.\n\nBy default, omits x-api-key, x-tenant-id, and x-service-key.\n\nExclusions (if provided) take precedence.\n\nEach value can be a raw string with an optional wildcard.",
      "type": "object",
      "properties": {
        "excludes": {
          "anyOf": [
            {
              "type": "array",
              "items": {
                "type": "string"
              }
            },
            {
              "type": "null"
            }
          ],
          "description": "Headers to exclude. Applied before the 'includes' checks.\n"
        },
        "includes": {
          "anyOf": [
            {
              "type": "array",
              "items": {
                "type": "string"
              }
            },
            {
              "type": "null"
            }
          ],
          "description": "Headers to include (if not also matches against an 'exludes' pattern.\n"
        }
      },
      "required": [
        "excludes",
        "includes"
      ]
    },
    "CorsConfig": {
      "title": "CorsConfig",
      "description": "Specifies Cross-Origin Resource Sharing (CORS) rules for your server.\n\nIf omitted, defaults are typically very restrictive (often no cross-origin requests).\nConfigure carefully if you want to allow usage from browsers hosted on other domains.",
      "type": "object",
      "properties": {
        "allow_credentials": {
          "type": "boolean",
          "description": "Optional. If `True`, cross-origin requests can include credentials (cookies, auth headers).\n\nDefault False to avoid accidentally exposing secured endpoints to untrusted sites.\n"
        },
        "allow_headers": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "description": "Optional. HTTP headers that can be used in cross-origin requests (e.g. [\"Content-Type\", \"Authorization\"])."
        },
        "allow_methods": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "description": "Optional. HTTP methods permitted for cross-origin requests (e.g. [\"GET\", \"POST\"]).\n\nDefault might be [\"GET\", \"POST\", \"OPTIONS\"] depending on your server framework.\n"
        },
        "allow_origin_regex": {
          "type": "string",
          "description": "Optional. A regex pattern for matching allowed origins, used if you have dynamic subdomains.\n"
        },
        "allow_origins": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "description": "Optional. List of allowed origins (e.g., \"https://example.com\").\n\nDefault is often an empty list (no external origins).\nUse \"*\" only if you trust all origins, as that bypasses most restrictions.\n"
        },
        "expose_headers": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "description": "Optional. List of headers that browsers are allowed to read from the response in cross-origin contexts."
        },
        "max_age": {
          "type": "integer",
          "description": "Optional. How many seconds the browser may cache preflight responses.\n\nDefault might be 600 (10 minutes). Larger values reduce preflight requests but can cause stale configurations.\n"
        }
      },
      "required": []
    },
    "StoreConfig": {
      "title": "StoreConfig",
      "description": "Configuration for the built-in long-term memory store.\n\nThis store can optionally perform semantic search. If you omit `index`,\nthe store will just handle traditional (non-embedded) data without vector lookups.",
      "type": "object",
      "properties": {
        "index": {
          "anyOf": [
            {
              "$ref": "#/$defs/IndexConfig"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines the vector-based semantic search configuration.\n\n- Generate embeddings according to `index.embed`\n- Enforce the embedding dimension given by `index.dims`\n- Embed only specified JSON fields (if any) from `index.fields`\n\nIf omitted, no vector index is initialized.\n"
        },
        "ttl": {
          "anyOf": [
            {
              "$ref": "#/$defs/TTLConfig"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines the TTL (time-to-live) behavior configuration.\n\nIf provided, the store will apply TTL settings according to the configuration.\nIf omitted, no TTL behavior is configured.\n"
        }
      },
      "required": []
    },
    "IndexConfig": {
      "title": "IndexConfig",
      "description": "Configuration for indexing documents for semantic search in the store.\n\nThis governs how text is converted into embeddings and stored for vector-based lookups.",
      "type": "object",
      "properties": {
        "dims": {
          "type": "integer",
          "description": "Required. Dimensionality of the embedding vectors you will store.\n\nMust match the output dimension of your selected embedding model or custom embed function.\nIf mismatched, you will likely encounter shape/size errors when inserting or querying vectors.\n\n"
        },
        "embed": {
          "type": "string",
          "description": "Required. Identifier or reference to the embedding model or a custom embedding function.\n\n- \"my_custom_embed\" if it's a known alias in your system\n"
        },
        "fields": {
          "anyOf": [
            {
              "type": "array",
              "items": {
                "type": "string"
              }
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. List of JSON fields to extract before generating embeddings.\n\nDefaults to [\"$\"], which means the entire JSON object is embedded as one piece of text.\nIf you provide multiple fields (e.g. [\"title\", \"content\"]), each is extracted and embedded separately,\noften saving token usage if you only care about certain parts of the data.\n"
        }
      },
      "required": []
    },
    "TTLConfig": {
      "title": "TTLConfig",
      "description": "Configuration for TTL (time-to-live) behavior in the store.",
      "type": "object",
      "properties": {
        "default_ttl": {
          "anyOf": [
            {
              "type": "number"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Default TTL (time-to-live) in minutes for new items.\n\nIf provided, all new items will have this TTL unless explicitly overridden.\nIf omitted, items will have no TTL by default.\n"
        },
        "refresh_on_read": {
          "type": "boolean",
          "description": "Default behavior for refreshing TTLs on read operations (`GET` and `SEARCH`).\n\nIf `True`, TTLs will be refreshed on read operations (get/search) by default.\nThis can be overridden per-operation by explicitly setting `refresh_ttl`.\nDefaults to `True` if not configured.\n"
        },
        "sweep_interval_minutes": {
          "anyOf": [
            {
              "type": "integer"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Interval in minutes between TTL sweep iterations.\n\nIf provided, the store will periodically delete expired items based on the TTL.\nIf omitted, no automatic sweeping will occur.\n"
        }
      },
      "required": []
    }
  },
  "title": "LangGraph CLI Configuration",
  "description": "Configuration schema for langgraph-cli",
  "version": "v0"
}
```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/schemas/schema.v0.json
```json
{
  "$ref": "#/$defs/Config",
  "$defs": {
    "Config": {
      "title": "Config",
      "description": "Top-level config for langgraph-cli or similar deployment tooling.",
      "type": "object",
      "required": [],
      "oneOf": [
        {
          "type": "object",
          "properties": {
            "python_version": {
              "type": "string",
              "description": "Optional. Python version in 'major.minor' format (e.g. '3.11').\nMust be at least 3.11 or greater for this deployment to function properly.\n",
              "enum": [
                "3.11",
                "3.12",
                "3.13"
              ]
            },
            "pip_config_file": {
              "anyOf": [
                {
                  "type": "string"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Path to a pip config file (e.g., \"/etc/pip.conf\" or \"pip.ini\") for controlling\npackage installation (custom indices, credentials, etc.).\n\nOnly relevant if Python dependencies are installed via pip. If omitted, default pip settings are used.\n"
            },
            "_INTERNAL_docker_tag": {
              "anyOf": [
                {
                  "type": "string"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Internal use only.\n"
            },
            "api_version": {
              "anyOf": [
                {
                  "type": "string"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Which semantic version of the LangGraph API server to use.\n\nDefaults to latest. Check the\nfor more information.\n"
            },
            "auth": {
              "anyOf": [
                {
                  "$ref": "#/$defs/AuthConfig"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Custom authentication config, including the path to your Python auth logic and\nthe OpenAPI security definitions it uses.\n"
            },
            "base_image": {
              "anyOf": [
                {
                  "type": "string"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Base image to use for the LangGraph API server.\n\nDefaults to langchain/langgraph-api or langchain/langgraphjs-api.\n"
            },
            "checkpointer": {
              "anyOf": [
                {
                  "$ref": "#/$defs/CheckpointerConfig"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Configuration for the built-in checkpointer, which handles checkpointing of state.\n\nIf omitted, no checkpointer is set up (the object store will still be present, however).\n"
            },
            "dependencies": {
              "type": "array",
              "items": {
                "type": "string"
              },
              "description": "List of Python dependencies to install, either from PyPI or local paths.\n"
            },
            "dockerfile_lines": {
              "type": "array",
              "items": {
                "type": "string"
              },
              "description": "Optional. Additional Docker instructions that will be appended to your base Dockerfile.\n\nUseful for installing OS packages, setting environment variables, etc."
            },
            "env": {
              "anyOf": [
                {
                  "type": "object",
                  "additionalProperties": {
                    "type": "string"
                  }
                },
                {
                  "type": "string"
                }
              ],
              "description": "Optional. Environment variables to set for your deployment.\n\n- If given as a dict, keys are variable names and values are their values.\n- If given as a string, it must be a path to a file containing lines in KEY=VALUE format.\n\nenv=\".env\n"
            },
            "graphs": {
              "type": "object",
              "additionalProperties": {
                "type": "string"
              },
              "description": "Optional. Named definitions of graphs, each pointing to a Python object.\n\n\nGraphs can be StateGraph, @entrypoint, or any other Pregel object OR they can point to (async) context\nmanagers that accept a single configuration argument (of type RunnableConfig) and return a pregel object\n(instance of Stategraph, etc.).\n"
            },
            "http": {
              "anyOf": [
                {
                  "$ref": "#/$defs/HttpConfig"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Configuration for the built-in HTTP server, controlling which custom routes are exposed\nand how cross-origin requests are handled.\n"
            },
            "image_distro": {
              "anyOf": [
                {
                  "enum": [
                    "bookworm",
                    "bullseye",
                    "debian",
                    "wolfi"
                  ]
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Linux distribution for the base image.\n\nMust be one of 'wolfi', 'debian', 'bullseye', or 'bookworm'.\nIf omitted, defaults to 'debian' ('latest').\n"
            },
            "keep_pkg_tools": {
              "anyOf": [
                {
                  "type": "boolean"
                },
                {
                  "type": "array",
                  "items": {
                    "type": "string"
                  }
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Control whether to retain Python packaging tools in the final image.\n\nYou can also set to true to include all packaging tools.\n"
            },
            "pip_installer": {
              "anyOf": [
                {
                  "type": "string"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Python package installer to use ('auto', 'pip', 'uv').\n\n"
            },
            "store": {
              "anyOf": [
                {
                  "$ref": "#/$defs/StoreConfig"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Configuration for the built-in long-term memory store, including semantic search indexing.\n\nIf omitted, no vector index is set up (the object store will still be present, however).\n"
            },
            "ui": {
              "anyOf": [
                {
                  "type": "object",
                  "additionalProperties": {
                    "type": "string"
                  }
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Named definitions of UI components emitted by the agent, each pointing to a JS/TS file.\n"
            }
          },
          "required": [
            "dependencies",
            "graphs"
          ]
        },
        {
          "type": "object",
          "properties": {
            "node_version": {
              "anyOf": [
                {
                  "type": "string",
                  "enum": [
                    "20"
                  ]
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Node.js version as a major version (e.g. '20'), if your deployment needs Node.\nMust be >= 20 if provided.\n"
            },
            "_INTERNAL_docker_tag": {
              "anyOf": [
                {
                  "type": "string"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Internal use only.\n"
            },
            "api_version": {
              "anyOf": [
                {
                  "type": "string"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Which semantic version of the LangGraph API server to use.\n\nDefaults to latest. Check the\nfor more information.\n"
            },
            "auth": {
              "anyOf": [
                {
                  "$ref": "#/$defs/AuthConfig"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Custom authentication config, including the path to your Python auth logic and\nthe OpenAPI security definitions it uses.\n"
            },
            "base_image": {
              "anyOf": [
                {
                  "type": "string"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Base image to use for the LangGraph API server.\n\nDefaults to langchain/langgraph-api or langchain/langgraphjs-api.\n"
            },
            "checkpointer": {
              "anyOf": [
                {
                  "$ref": "#/$defs/CheckpointerConfig"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Configuration for the built-in checkpointer, which handles checkpointing of state.\n\nIf omitted, no checkpointer is set up (the object store will still be present, however).\n"
            },
            "dependencies": {
              "type": "array",
              "items": {
                "type": "string"
              },
              "description": "List of Python dependencies to install, either from PyPI or local paths.\n"
            },
            "dockerfile_lines": {
              "type": "array",
              "items": {
                "type": "string"
              },
              "description": "Optional. Additional Docker instructions that will be appended to your base Dockerfile.\n\nUseful for installing OS packages, setting environment variables, etc."
            },
            "env": {
              "anyOf": [
                {
                  "type": "object",
                  "additionalProperties": {
                    "type": "string"
                  }
                },
                {
                  "type": "string"
                }
              ],
              "description": "Optional. Environment variables to set for your deployment.\n\n- If given as a dict, keys are variable names and values are their values.\n- If given as a string, it must be a path to a file containing lines in KEY=VALUE format.\n\nenv=\".env\n"
            },
            "graphs": {
              "type": "object",
              "additionalProperties": {
                "type": "string"
              },
              "description": "Optional. Named definitions of graphs, each pointing to a Python object.\n\n\nGraphs can be StateGraph, @entrypoint, or any other Pregel object OR they can point to (async) context\nmanagers that accept a single configuration argument (of type RunnableConfig) and return a pregel object\n(instance of Stategraph, etc.).\n"
            },
            "http": {
              "anyOf": [
                {
                  "$ref": "#/$defs/HttpConfig"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Configuration for the built-in HTTP server, controlling which custom routes are exposed\nand how cross-origin requests are handled.\n"
            },
            "image_distro": {
              "anyOf": [
                {
                  "type": "string",
                  "enum": [
                    "debian",
                    "wolfi"
                  ]
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Linux distribution for the base image.\n\nMust be one of 'wolfi', 'debian', 'bullseye', or 'bookworm'.\nIf omitted, defaults to 'debian' ('latest').\n"
            },
            "keep_pkg_tools": {
              "anyOf": [
                {
                  "type": "boolean"
                },
                {
                  "type": "array",
                  "items": {
                    "type": "string"
                  }
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Control whether to retain Python packaging tools in the final image.\n\nYou can also set to true to include all packaging tools.\n"
            },
            "pip_installer": {
              "anyOf": [
                {
                  "type": "string"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Python package installer to use ('auto', 'pip', 'uv').\n\n"
            },
            "store": {
              "anyOf": [
                {
                  "$ref": "#/$defs/StoreConfig"
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Configuration for the built-in long-term memory store, including semantic search indexing.\n\nIf omitted, no vector index is set up (the object store will still be present, however).\n"
            },
            "ui": {
              "anyOf": [
                {
                  "type": "object",
                  "additionalProperties": {
                    "type": "string"
                  }
                },
                {
                  "type": "null"
                }
              ],
              "description": "Optional. Named definitions of UI components emitted by the agent, each pointing to a JS/TS file.\n"
            }
          },
          "required": [
            "node_version",
            "graphs"
          ]
        }
      ]
    },
    "AuthConfig": {
      "title": "AuthConfig",
      "description": "Configuration for custom authentication logic and how it integrates into the OpenAPI spec.",
      "type": "object",
      "properties": {
        "disable_studio_auth": {
          "type": "boolean",
          "description": "Optional. Whether to disable LangSmith API-key authentication for requests originating the Studio.\n\nDefaults to False, meaning that if a particular header is set, the server will verify the `x-api-key` header\nvalue is a valid API key for the deployment's workspace. If `True`, all requests will go through your custom\nauthentication logic, regardless of origin of the request.\n"
        },
        "openapi": {
          "$ref": "#/$defs/SecurityConfig",
          "description": "The security configuration to include in your server's OpenAPI spec.\n\n{\n}\n}\n}\n},\n]\n}\n"
        },
        "path": {
          "type": "string",
          "description": "Required. Path to an instance of the Auth() class that implements custom authentication.\n"
        }
      },
      "required": []
    },
    "SecurityConfig": {
      "title": "SecurityConfig",
      "description": "Configuration for OpenAPI security definitions and requirements.\n\nUseful for specifying global or path-level authentication and authorization flows\n(e.g., OAuth2, API key headers, etc.).",
      "type": "object",
      "properties": {
        "paths": {
          "type": "object",
          "additionalProperties": {
            "type": "object",
            "additionalProperties": {
              "type": "array",
              "items": {
                "type": "object",
                "additionalProperties": {
                  "type": "array",
                  "items": {
                    "type": "string"
                  }
                }
              }
            }
          },
          "description": "Path-specific security overrides.\n\n- Keys that are HTTP methods (e.g., \"GET\", \"POST\"),\n- Values are lists of security definitions (just like `security`) for that method.\n"
        },
        "security": {
          "type": "array",
          "items": {
            "type": "object",
            "additionalProperties": {
              "type": "array",
              "items": {
                "type": "string"
              }
            }
          },
          "description": "Global security requirements across all endpoints.\n\nEach element in the list maps a security scheme (e.g. \"OAuth2\") to a list of scopes (e.g. [\"read\", \"write\"])."
        },
        "securitySchemes": {
          "type": "object",
          "additionalProperties": {
            "type": "object"
          },
          "description": "Describe each security scheme recognized by your OpenAPI spec.\n\nKeys are scheme names (e.g. \"OAuth2\", \"ApiKeyAuth\") and values are their definitions."
        }
      },
      "required": []
    },
    "CheckpointerConfig": {
      "title": "CheckpointerConfig",
      "description": "Configuration for the built-in checkpointer, which handles checkpointing of state.\n\nIf omitted, no checkpointer is set up (the object store will still be present, however).",
      "type": "object",
      "properties": {
        "serde": {
          "anyOf": [
            {
              "$ref": "#/$defs/SerdeConfig"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines the serde configuration.\n\nIf provided, the checkpointer will apply serde settings according to the configuration.\nIf omitted, no serde behavior is configured.\n\nThis configuration requires server version 0.5 or later to take effect.\n"
        },
        "ttl": {
          "anyOf": [
            {
              "$ref": "#/$defs/ThreadTTLConfig"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines the TTL (time-to-live) behavior configuration.\n\nIf provided, the checkpointer will apply TTL settings according to the configuration.\nIf omitted, no TTL behavior is configured.\n"
        }
      },
      "required": []
    },
    "SerdeConfig": {
      "title": "SerdeConfig",
      "description": "Configuration for the built-in serde, which handles checkpointing of state.\n\nIf omitted, no serde is set up (the object store will still be present, however).",
      "type": "object",
      "properties": {
        "allowed_json_modules": {
          "anyOf": [
            {
              "type": "array",
              "items": {
                "type": "array",
                "items": {
                  "type": "string"
                }
              }
            },
            {
              "type": "boolean"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. List of allowed python modules to de-serialize custom objects from.\n\nIf provided, only the specified modules will be allowed to be deserialized.\nIf omitted, no modules are allowed, and the object returned will simply be a json object OR\na deserialized langchain object.\n"
        },
        "pickle_fallback": {
          "type": "boolean",
          "description": "Optional. Whether to allow pickling as a fallback for deserialization.\n\nIf True, pickling will be allowed as a fallback for deserialization.\nIf False, pickling will not be allowed as a fallback for deserialization.\nDefaults to True if not configured."
        }
      },
      "required": []
    },
    "ThreadTTLConfig": {
      "title": "ThreadTTLConfig",
      "description": "Configure a default TTL for checkpointed data within threads.",
      "type": "object",
      "properties": {
        "default_ttl": {
          "anyOf": [
            {
              "type": "number"
            },
            {
              "type": "null"
            }
          ],
          "description": "Default TTL (time-to-live) in minutes for checkpointed data."
        },
        "strategy": {
          "enum": [
            "delete"
          ],
          "description": "Strategy to use for deleting checkpointed data.\n"
        },
        "sweep_interval_minutes": {
          "anyOf": [
            {
              "type": "integer"
            },
            {
              "type": "null"
            }
          ],
          "description": "Interval in minutes between sweep iterations.\nIf omitted, a default interval will be used (typically ~ 5 minutes)."
        }
      },
      "required": []
    },
    "HttpConfig": {
      "title": "HttpConfig",
      "description": "Configuration for the built-in HTTP server that powers your deployment's routes and endpoints.",
      "type": "object",
      "properties": {
        "app": {
          "type": "string",
          "description": "Optional. Import path to a custom Starlette/FastAPI application to mount.\n"
        },
        "configurable_headers": {
          "anyOf": [
            {
              "$ref": "#/$defs/ConfigurableHeaderConfig"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines how headers are treated for a run's configuration.\n\nYou can include or exclude headers as configurable values to condition your\nagent's behavior or permissions on a request's headers."
        },
        "cors": {
          "anyOf": [
            {
              "$ref": "#/$defs/CorsConfig"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines CORS restrictions. If omitted, no special rules are set and\ncross-origin behavior depends on default server settings.\n"
        },
        "disable_assistants": {
          "type": "boolean",
          "description": "Optional. If `True`, /assistants routes are removed from the server.\n\nDefault is False (meaning /assistants is enabled).\n"
        },
        "disable_mcp": {
          "type": "boolean",
          "description": "Optional. If `True`, /mcp routes are removed, disabling the MCP server.\n\nDefault is False.\n"
        },
        "disable_meta": {
          "type": "boolean",
          "description": "Optional. Remove meta endpoints.\n\n\nDefault is False.\n"
        },
        "disable_runs": {
          "type": "boolean",
          "description": "Optional. If `True`, /runs routes are removed.\n\nDefault is False.\n"
        },
        "disable_store": {
          "type": "boolean",
          "description": "Optional. If `True`, /store routes are removed, disabling direct store interactions via HTTP.\n\nDefault is False.\n"
        },
        "disable_threads": {
          "type": "boolean",
          "description": "Optional. If `True`, /threads routes are removed.\n\nDefault is False.\n"
        },
        "enable_custom_route_auth": {
          "type": "boolean",
          "description": "Optional. If `True`, authentication is enabled for custom routes,\nnot just the routes that are protected by default.\n(Routes protected by default include /assistants, /threads, and /runs).\n\nDefault is False. This flag only affects authentication behavior\nif `app` is provided and contains custom routes.\n"
        },
        "logging_headers": {
          "anyOf": [
            {
              "$ref": "#/$defs/ConfigurableHeaderConfig"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines which headers are excluded from logging."
        },
        "middleware_order": {
          "anyOf": [
            {
              "enum": [
                "auth_first",
                "middleware_first"
              ]
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines the order in which to apply server customizations.\n"
        }
      },
      "required": []
    },
    "ConfigurableHeaderConfig": {
      "title": "ConfigurableHeaderConfig",
      "description": "Customize which headers to include as configurable values in your runs.\n\nBy default, omits x-api-key, x-tenant-id, and x-service-key.\n\nExclusions (if provided) take precedence.\n\nEach value can be a raw string with an optional wildcard.",
      "type": "object",
      "properties": {
        "excludes": {
          "anyOf": [
            {
              "type": "array",
              "items": {
                "type": "string"
              }
            },
            {
              "type": "null"
            }
          ],
          "description": "Headers to exclude. Applied before the 'includes' checks.\n"
        },
        "includes": {
          "anyOf": [
            {
              "type": "array",
              "items": {
                "type": "string"
              }
            },
            {
              "type": "null"
            }
          ],
          "description": "Headers to include (if not also matches against an 'exludes' pattern.\n"
        }
      },
      "required": [
        "excludes",
        "includes"
      ]
    },
    "CorsConfig": {
      "title": "CorsConfig",
      "description": "Specifies Cross-Origin Resource Sharing (CORS) rules for your server.\n\nIf omitted, defaults are typically very restrictive (often no cross-origin requests).\nConfigure carefully if you want to allow usage from browsers hosted on other domains.",
      "type": "object",
      "properties": {
        "allow_credentials": {
          "type": "boolean",
          "description": "Optional. If `True`, cross-origin requests can include credentials (cookies, auth headers).\n\nDefault False to avoid accidentally exposing secured endpoints to untrusted sites.\n"
        },
        "allow_headers": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "description": "Optional. HTTP headers that can be used in cross-origin requests (e.g. [\"Content-Type\", \"Authorization\"])."
        },
        "allow_methods": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "description": "Optional. HTTP methods permitted for cross-origin requests (e.g. [\"GET\", \"POST\"]).\n\nDefault might be [\"GET\", \"POST\", \"OPTIONS\"] depending on your server framework.\n"
        },
        "allow_origin_regex": {
          "type": "string",
          "description": "Optional. A regex pattern for matching allowed origins, used if you have dynamic subdomains.\n"
        },
        "allow_origins": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "description": "Optional. List of allowed origins (e.g., \"https://example.com\").\n\nDefault is often an empty list (no external origins).\nUse \"*\" only if you trust all origins, as that bypasses most restrictions.\n"
        },
        "expose_headers": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "description": "Optional. List of headers that browsers are allowed to read from the response in cross-origin contexts."
        },
        "max_age": {
          "type": "integer",
          "description": "Optional. How many seconds the browser may cache preflight responses.\n\nDefault might be 600 (10 minutes). Larger values reduce preflight requests but can cause stale configurations.\n"
        }
      },
      "required": []
    },
    "StoreConfig": {
      "title": "StoreConfig",
      "description": "Configuration for the built-in long-term memory store.\n\nThis store can optionally perform semantic search. If you omit `index`,\nthe store will just handle traditional (non-embedded) data without vector lookups.",
      "type": "object",
      "properties": {
        "index": {
          "anyOf": [
            {
              "$ref": "#/$defs/IndexConfig"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines the vector-based semantic search configuration.\n\n- Generate embeddings according to `index.embed`\n- Enforce the embedding dimension given by `index.dims`\n- Embed only specified JSON fields (if any) from `index.fields`\n\nIf omitted, no vector index is initialized.\n"
        },
        "ttl": {
          "anyOf": [
            {
              "$ref": "#/$defs/TTLConfig"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Defines the TTL (time-to-live) behavior configuration.\n\nIf provided, the store will apply TTL settings according to the configuration.\nIf omitted, no TTL behavior is configured.\n"
        }
      },
      "required": []
    },
    "IndexConfig": {
      "title": "IndexConfig",
      "description": "Configuration for indexing documents for semantic search in the store.\n\nThis governs how text is converted into embeddings and stored for vector-based lookups.",
      "type": "object",
      "properties": {
        "dims": {
          "type": "integer",
          "description": "Required. Dimensionality of the embedding vectors you will store.\n\nMust match the output dimension of your selected embedding model or custom embed function.\nIf mismatched, you will likely encounter shape/size errors when inserting or querying vectors.\n\n"
        },
        "embed": {
          "type": "string",
          "description": "Required. Identifier or reference to the embedding model or a custom embedding function.\n\n- \"my_custom_embed\" if it's a known alias in your system\n"
        },
        "fields": {
          "anyOf": [
            {
              "type": "array",
              "items": {
                "type": "string"
              }
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. List of JSON fields to extract before generating embeddings.\n\nDefaults to [\"$\"], which means the entire JSON object is embedded as one piece of text.\nIf you provide multiple fields (e.g. [\"title\", \"content\"]), each is extracted and embedded separately,\noften saving token usage if you only care about certain parts of the data.\n"
        }
      },
      "required": []
    },
    "TTLConfig": {
      "title": "TTLConfig",
      "description": "Configuration for TTL (time-to-live) behavior in the store.",
      "type": "object",
      "properties": {
        "default_ttl": {
          "anyOf": [
            {
              "type": "number"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Default TTL (time-to-live) in minutes for new items.\n\nIf provided, all new items will have this TTL unless explicitly overridden.\nIf omitted, items will have no TTL by default.\n"
        },
        "refresh_on_read": {
          "type": "boolean",
          "description": "Default behavior for refreshing TTLs on read operations (`GET` and `SEARCH`).\n\nIf `True`, TTLs will be refreshed on read operations (get/search) by default.\nThis can be overridden per-operation by explicitly setting `refresh_ttl`.\nDefaults to `True` if not configured.\n"
        },
        "sweep_interval_minutes": {
          "anyOf": [
            {
              "type": "integer"
            },
            {
              "type": "null"
            }
          ],
          "description": "Optional. Interval in minutes between TTL sweep iterations.\n\nIf provided, the store will periodically delete expired items based on the TTL.\nIf omitted, no automatic sweeping will occur.\n"
        }
      },
      "required": []
    }
  },
  "title": "LangGraph CLI Configuration",
  "description": "Configuration schema for langgraph-cli",
  "version": "v0"
}
```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/schemas/version.schema.json
```json
{
  "$id": "https://github.com/langchain-ai/langgraph/libs/cli/schemas/version.schema.json",
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "LangGraph Platform configuration (when building via the langgraph-cli).",
  "type": "object",
  "oneOf": [
    {
      "allOf": [
        {
          "oneOf": [
            {
              "properties": {
                "version": {
                  "type": "string",
                  "maxLength": 0
                }
              },
              "required": ["version"]
            },
            {
              "not": {
                "required": ["version"]
              }
            }
          ]
        },
        {
          "$ref": "https://raw.githubusercontent.com/langchain-ai/langgraph/main/libs/cli/schemas/schema.json"
        }
      ]
    },
    {
      "allOf": [
        {
          "properties": {
            "version": { "const": "v0" }
          },
          "required": ["version"]
        },
        {
          "$ref": "https://raw.githubusercontent.com/langchain-ai/langgraph/main/libs/cli/schemas/schema.v0.json"
        }
      ]
    }
  ]
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/__init__.py
```py

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/__init__.py
```py

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/agent.py
```py
import asyncio
import os
from collections.abc import Sequence
from typing import Annotated, TypedDict

from langchain_core.language_models.fake_chat_models import FakeListChatModel
from langchain_core.messages import BaseMessage, HumanMessage, ToolMessage
from langgraph.graph import END, StateGraph, add_messages

# check that env var is present
os.environ["SOME_ENV_VAR"]


class AgentState(TypedDict):
    some_bytes: bytes
    some_byte_array: bytearray
    dict_with_bytes: dict[str, bytes]
    messages: Annotated[Sequence[BaseMessage], add_messages]
    sleep: int


async def call_model(state, config):
    if sleep := state.get("sleep"):
        await asyncio.sleep(sleep)

    messages = state["messages"]

    if len(messages) > 1:
        assert state["some_bytes"] == b"some_bytes"
        assert state["some_byte_array"] == bytearray(b"some_byte_array")
        assert state["dict_with_bytes"] == {"more_bytes": b"more_bytes"}

    # hacky way to reset model to the "first" response
    if isinstance(messages[-1], HumanMessage):
        model.i = 0

    response = await model.ainvoke(messages)
    return {
        "messages": [response],
        "some_bytes": b"some_bytes",
        "some_byte_array": bytearray(b"some_byte_array"),
        "dict_with_bytes": {"more_bytes": b"more_bytes"},
    }


def call_tool(state):
    last_message_content = state["messages"][-1].content
    return {
        "messages": [
            ToolMessage(
                f"tool_call__{last_message_content}", tool_call_id="tool_call_id"
            )
        ]
    }


def should_continue(state):
    messages = state["messages"]
    last_message = messages[-1]
    if last_message.content == "end":
        return END
    else:
        return "tool"


# NOTE: the model cycles through responses infinitely here
model = FakeListChatModel(responses=["begin", "end"])
workflow = StateGraph(AgentState)

workflow.add_node("agent", call_model)
workflow.add_node("tool", call_tool)

workflow.set_entry_point("agent")

workflow.add_conditional_edges(
    "agent",
    should_continue,
)

workflow.add_edge("tool", "agent")

graph = workflow.compile()

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/conftest.py
```py
import os
from unittest.mock import patch

import pytest


@pytest.fixture(autouse=True)
def disable_analytics_env() -> None:
    """Disable analytics for unit tests LANGGRAPH_CLI_NO_ANALYTICS."""
    # First check if the environment variable is already set, if so, log a warning prior
    # to overriding it.
    if "LANGGRAPH_CLI_NO_ANALYTICS" in os.environ:
        print("⚠️ LANGGRAPH_CLI_NO_ANALYTICS is set. Overriding it for the test.")

    with patch.dict(os.environ, {"LANGGRAPH_CLI_NO_ANALYTICS": "0"}):
        yield

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/helpers.py
```py
def clean_empty_lines(input_str: str):
    return "\n".join(filter(None, input_str.splitlines()))

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/pipconfig.txt
```txt

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/test_config.json
```json
{
    "python_version": "3.12",
    "pip_config_file": "pipconfig.txt",
    "dockerfile_lines": [
        "ARG meow=woof"
    ],
    "dependencies": [
        "langchain_openai",
        "starlette",
        "."
    ],
    "graphs": {
        "agent": "graphs/agent.py:graph"
    },
    "env": ".env",
    "http": {
        "app": "../../examples/my_app.py:app" 
    }
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/test_config.py
```py
import copy
import json
import os
import pathlib
import tempfile
import textwrap

import click
import pytest

from langgraph_cli.config import (
    _BUILD_TOOLS,
    _get_pip_cleanup_lines,
    config_to_compose,
    config_to_docker,
    docker_tag,
    validate_config,
    validate_config_file,
)
from langgraph_cli.util import clean_empty_lines

FORMATTED_CLEANUP_LINES = _get_pip_cleanup_lines(
    install_cmd="uv pip install --system",
    to_uninstall=("pip", "setuptools", "wheel"),
    pip_installer="uv",
)

PATH_TO_CONFIG = pathlib.Path(__file__).parent / "test_config.json"


def test_validate_config():
    # minimal config
    expected_config = {
        "dependencies": ["."],
        "graphs": {
            "agent": "./agent.py:graph",
        },
    }
    actual_config = validate_config(expected_config)
    expected_config = {
        "base_image": None,
        "python_version": "3.11",
        "node_version": None,
        "pip_config_file": None,
        "pip_installer": "auto",
        "image_distro": "debian",
        "dockerfile_lines": [],
        "env": {},
        "store": None,
        "auth": None,
        "checkpointer": None,
        "http": None,
        "ui": None,
        "ui_config": None,
        "keep_pkg_tools": None,
        **expected_config,
    }
    assert actual_config == expected_config

    # full config
    env = ".env"
    expected_config = {
        "base_image": None,
        "python_version": "3.12",
        "node_version": None,
        "pip_config_file": "pipconfig.txt",
        "pip_installer": "auto",
        "image_distro": "debian",
        "dockerfile_lines": ["ARG meow"],
        "dependencies": [".", "langchain"],
        "graphs": {
            "agent": "./agent.py:graph",
        },
        "env": env,
        "store": None,
        "auth": None,
        "checkpointer": None,
        "http": None,
        "ui": None,
        "ui_config": None,
        "keep_pkg_tools": None,
    }
    actual_config = validate_config(expected_config)
    assert actual_config == expected_config
    expected_config["python_version"] = "3.13"
    actual_config = validate_config(expected_config)
    assert actual_config == expected_config

    # check wrong python version raises
    with pytest.raises(click.UsageError):
        validate_config({"python_version": "3.9"})

    # check missing dependencies key raises
    with pytest.raises(click.UsageError):
        validate_config(
            {"python_version": "3.9", "graphs": {"agent": "./agent.py:graph"}}
        )

    # check missing graphs key raises
    with pytest.raises(click.UsageError):
        validate_config({"python_version": "3.9", "dependencies": ["."]})

    with pytest.raises(click.UsageError) as exc_info:
        validate_config({"python_version": "3.11.0"})
    assert "Invalid Python version format" in str(exc_info.value)

    with pytest.raises(click.UsageError) as exc_info:
        validate_config({"python_version": "3"})
    assert "Invalid Python version format" in str(exc_info.value)

    with pytest.raises(click.UsageError) as exc_info:
        validate_config({"python_version": "abc.def"})
    assert "Invalid Python version format" in str(exc_info.value)

    with pytest.raises(click.UsageError) as exc_info:
        validate_config({"python_version": "3.10"})
    assert "Minimum required version" in str(exc_info.value)

    config = validate_config(
        {
            "python_version": "3.11-bullseye",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
        }
    )
    assert config["python_version"] == "3.11-bullseye"

    config = validate_config(
        {
            "python_version": "3.12-slim",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
        }
    )
    assert config["python_version"] == "3.12-slim"
    with pytest.raises(ValueError, match="Invalid http.app format"):
        validate_config(
            {
                "python_version": "3.12",
                "dependencies": ["."],
                "graphs": {"agent": "./agent.py:graph"},
                "http": {"app": "../../examples/my_app.py"},
            }
        )


def test_validate_config_image_distro():
    """Test validation of image_distro field."""
    # Valid image_distro values should work
    config = validate_config(
        {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
            "image_distro": "debian",
        }
    )
    assert config["image_distro"] == "debian"

    config = validate_config(
        {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
            "image_distro": "wolfi",
        }
    )
    assert config["image_distro"] == "wolfi"

    # Missing image_distro should default to 'debian'
    config = validate_config(
        {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
        }
    )
    assert config["image_distro"] == "debian"

    # Invalid image_distro values should raise error
    with pytest.raises(click.UsageError) as exc_info:
        validate_config(
            {
                "python_version": "3.11",
                "dependencies": ["."],
                "graphs": {"agent": "./agent.py:graph"},
                "image_distro": "ubuntu",
            }
        )
    assert "Invalid image_distro: 'ubuntu'" in str(exc_info.value)

    with pytest.raises(click.UsageError) as exc_info:
        validate_config(
            {
                "python_version": "3.11",
                "dependencies": ["."],
                "graphs": {"agent": "./agent.py:graph"},
                "image_distro": "alpine",
            }
        )
    assert "Invalid image_distro: 'alpine'" in str(exc_info.value)

    # Test base Node.js config with image distro
    config = validate_config(
        {
            "node_version": "20",
            "graphs": {"agent": "./agent.js:graph"},
            "image_distro": "wolfi",
        }
    )
    assert config["image_distro"] == "wolfi"

    # Test Node.js config with no distro specified
    config = validate_config(
        {
            "node_version": "20",
            "graphs": {"agent": "./agent.js:graph"},
        }
    )
    assert config["image_distro"] == "debian"


def test_validate_config_pip_installer():
    """Test validation of pip_installer field."""
    # Valid pip_installer values should work
    config = validate_config(
        {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
            "pip_installer": "auto",
        }
    )
    assert config["pip_installer"] == "auto"

    config = validate_config(
        {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
            "pip_installer": "pip",
        }
    )
    assert config["pip_installer"] == "pip"

    config = validate_config(
        {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
            "pip_installer": "uv",
        }
    )
    assert config["pip_installer"] == "uv"

    # Missing pip_installer should default to "auto"
    config = validate_config(
        {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
        }
    )
    assert config["pip_installer"] == "auto"

    # Invalid pip_installer values should raise error
    with pytest.raises(click.UsageError) as exc_info:
        validate_config(
            {
                "python_version": "3.11",
                "dependencies": ["."],
                "graphs": {"agent": "./agent.py:graph"},
                "pip_installer": "conda",
            }
        )
    assert "Invalid pip_installer: 'conda'" in str(exc_info.value)
    assert "Must be 'auto', 'pip', or 'uv'" in str(exc_info.value)

    with pytest.raises(click.UsageError) as exc_info:
        validate_config(
            {
                "python_version": "3.11",
                "dependencies": ["."],
                "graphs": {"agent": "./agent.py:graph"},
                "pip_installer": "invalid",
            }
        )
    assert "Invalid pip_installer: 'invalid'" in str(exc_info.value)


def test_validate_config_file():
    with tempfile.TemporaryDirectory() as tmpdir:
        tmpdir_path = pathlib.Path(tmpdir)

        config_path = tmpdir_path / "langgraph.json"

        node_config = {"node_version": "20", "graphs": {"agent": "./agent.js:graph"}}
        with open(config_path, "w") as f:
            json.dump(node_config, f)

        validate_config_file(config_path)

        package_json = {"name": "test", "engines": {"node": "20"}}
        with open(tmpdir_path / "package.json", "w") as f:
            json.dump(package_json, f)
        validate_config_file(config_path)

        package_json["engines"]["node"] = "20.18"
        with open(tmpdir_path / "package.json", "w") as f:
            json.dump(package_json, f)
        with pytest.raises(click.UsageError, match="Use major version only"):
            validate_config_file(config_path)

        package_json["engines"] = {"node": "18"}
        with open(tmpdir_path / "package.json", "w") as f:
            json.dump(package_json, f)
        with pytest.raises(click.UsageError, match="must be >= 20"):
            validate_config_file(config_path)

        package_json["engines"] = {"node": "20", "deno": "1.0"}
        with open(tmpdir_path / "package.json", "w") as f:
            json.dump(package_json, f)
        with pytest.raises(click.UsageError, match="Only 'node' engine is supported"):
            validate_config_file(config_path)

        with open(tmpdir_path / "package.json", "w") as f:
            f.write("{invalid json")
        with pytest.raises(click.UsageError, match="Invalid package.json"):
            validate_config_file(config_path)

        python_config = {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
        }
        with open(config_path, "w") as f:
            json.dump(python_config, f)

        validate_config_file(config_path)

        for package_content in [
            {"name": "test"},
            {"engines": {"node": "18"}},
            {"engines": {"node": "20", "deno": "1.0"}},
            "{invalid json",
        ]:
            with open(tmpdir_path / "package.json", "w") as f:
                if isinstance(package_content, dict):
                    json.dump(package_content, f)
                else:
                    f.write(package_content)
            validate_config_file(config_path)


def test_validate_config_multiplatform():
    # default node
    config = validate_config(
        {"dependencies": ["."], "graphs": {"js": "./js.mts:graph"}}
    )
    assert config["node_version"] == "20"
    assert config["python_version"] is None

    # default multiplatform
    config = validate_config(
        {
            "node_version": "22",
            "python_version": "3.12",
            "dependencies": ["."],
            "graphs": {"python": "./python.py:graph", "js": "./js.mts:graph"},
        }
    )
    assert config["node_version"] == "22"
    assert config["python_version"] == "3.12"

    # default multiplatform (full infer)
    graphs = {"python": "./python.py:graph", "js": "./js.mts:graph"}
    config = validate_config({"dependencies": ["."], "graphs": graphs})
    assert config["node_version"] == "20"
    assert config["python_version"] == "3.11"

    # default multiplatform (partial node)
    config = validate_config(
        {"node_version": "22", "dependencies": ["."], "graphs": graphs}
    )
    assert config["node_version"] == "22"
    assert config["python_version"] == "3.11"

    # default multiplatform (partial python)
    config = validate_config(
        {"python_version": "3.12", "dependencies": ["."], "graphs": graphs}
    )
    assert config["node_version"] == "20"
    assert config["python_version"] == "3.12"

    # no known extension (assumes python)
    config = validate_config(
        {
            "dependencies": ["./local", "./shared_utils"],
            "graphs": {"agent": "local.workflow:graph"},
            "env": ".env",
        }
    )
    assert config["node_version"] is None
    assert config["python_version"] == "3.11"


# config_to_docker
def test_config_to_docker_simple():
    graphs = {"agent": "./agent.py:graph"}
    actual_docker_stdin, additional_contexts = config_to_docker(
        PATH_TO_CONFIG,
        validate_config(
            {
                "dependencies": [".", "../../examples/graphs_reqs_a", "../../examples"],
                "graphs": graphs,
                "http": {"app": "../../examples/my_app.py:app"},
            }
        ),
        "langchain/langgraph-api",
    )
    expected_docker_stdin = f"""\
# syntax=docker/dockerfile:1.4
FROM langchain/langgraph-api:3.11
# -- Installing local requirements --
COPY --from=outer-requirements.txt requirements.txt /deps/outer-graphs_reqs_a/graphs_reqs_a/requirements.txt
RUN PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt -r /deps/outer-graphs_reqs_a/graphs_reqs_a/requirements.txt
# -- End of local requirements install --
# -- Adding local package ../../examples --
COPY --from=examples . /deps/examples
# -- End of local package ../../examples --
# -- Adding non-package dependency unit_tests --
ADD . /deps/outer-unit_tests/unit_tests
RUN set -ex && \\
    for line in '[project]' \\
                'name = "unit_tests"' \\
                'version = "0.1"' \\
                '[tool.setuptools.package-data]' \\
                '"*" = ["**/*"]' \\
                '[build-system]' \\
                'requires = ["setuptools>=61"]' \\
                'build-backend = "setuptools.build_meta"'; do \\
        echo "$line" >> /deps/outer-unit_tests/pyproject.toml; \\
    done
# -- End of non-package dependency unit_tests --
# -- Adding non-package dependency graphs_reqs_a --
COPY --from=outer-graphs_reqs_a . /deps/outer-graphs_reqs_a/graphs_reqs_a
RUN set -ex && \\
    for line in '[project]' \\
                'name = "graphs_reqs_a"' \\
                'version = "0.1"' \\
                '[tool.setuptools.package-data]' \\
                '"*" = ["**/*"]' \\
                '[build-system]' \\
                'requires = ["setuptools>=61"]' \\
                'build-backend = "setuptools.build_meta"'; do \\
        echo "$line" >> /deps/outer-graphs_reqs_a/pyproject.toml; \\
    done
# -- End of non-package dependency graphs_reqs_a --
# -- Installing all local dependencies --
RUN for dep in /deps/*; do             echo "Installing $dep";             if [ -d "$dep" ]; then                 echo "Installing $dep";                 (cd "$dep" && PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt -e .);             fi;         done
# -- End of local dependencies install --
ENV LANGGRAPH_HTTP='{{"app": "/deps/examples/my_app.py:app"}}'
ENV LANGSERVE_GRAPHS='{{"agent": "/deps/outer-unit_tests/unit_tests/agent.py:graph"}}'
{FORMATTED_CLEANUP_LINES}
WORKDIR /deps/outer-unit_tests/unit_tests\
"""
    assert clean_empty_lines(actual_docker_stdin) == expected_docker_stdin

    assert additional_contexts == {
        "outer-graphs_reqs_a": str(
            (pathlib.Path(__file__).parent / "../../examples/graphs_reqs_a").resolve()
        ),
        "examples": str((pathlib.Path(__file__).parent / "../../examples").resolve()),
    }


def test_config_to_docker_outside_path():
    graphs = {"agent": "./agent.py:graph"}
    actual_docker_stdin, additional_contexts = config_to_docker(
        PATH_TO_CONFIG,
        validate_config({"dependencies": [".", ".."], "graphs": graphs}),
        "langchain/langgraph-api",
    )
    expected_docker_stdin = (
        """\
# syntax=docker/dockerfile:1.4
FROM langchain/langgraph-api:3.11
# -- Adding non-package dependency unit_tests --
ADD . /deps/outer-unit_tests/unit_tests
RUN set -ex && \\
    for line in '[project]' \\
                'name = "unit_tests"' \\
                'version = "0.1"' \\
                '[tool.setuptools.package-data]' \\
                '"*" = ["**/*"]' \\
                '[build-system]' \\
                'requires = ["setuptools>=61"]' \\
                'build-backend = "setuptools.build_meta"'; do \\
        echo "$line" >> /deps/outer-unit_tests/pyproject.toml; \\
    done
# -- End of non-package dependency unit_tests --
# -- Adding non-package dependency tests --
COPY --from=outer-tests . /deps/outer-tests/tests
RUN set -ex && \\
    for line in '[project]' \\
                'name = "tests"' \\
                'version = "0.1"' \\
                '[tool.setuptools.package-data]' \\
                '"*" = ["**/*"]' \\
                '[build-system]' \\
                'requires = ["setuptools>=61"]' \\
                'build-backend = "setuptools.build_meta"'; do \\
        echo "$line" >> /deps/outer-tests/pyproject.toml; \\
    done
# -- End of non-package dependency tests --
# -- Installing all local dependencies --
RUN for dep in /deps/*; do             echo "Installing $dep";             if [ -d "$dep" ]; then                 echo "Installing $dep";                 (cd "$dep" && PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt -e .);             fi;         done
# -- End of local dependencies install --
ENV LANGSERVE_GRAPHS='{"agent": "/deps/outer-unit_tests/unit_tests/agent.py:graph"}'
"""
        + FORMATTED_CLEANUP_LINES
        + """
WORKDIR /deps/outer-unit_tests/unit_tests\
"""
    )
    assert clean_empty_lines(actual_docker_stdin) == expected_docker_stdin
    assert additional_contexts == {
        "outer-tests": str(pathlib.Path(__file__).parent.parent.absolute()),
    }


def test_config_to_docker_pipconfig():
    graphs = {"agent": "./agent.py:graph"}
    actual_docker_stdin, additional_contexts = config_to_docker(
        PATH_TO_CONFIG,
        validate_config(
            {
                "dependencies": ["."],
                "graphs": graphs,
                "pip_config_file": "pipconfig.txt",
            }
        ),
        "langchain/langgraph-api",
    )
    expected_docker_stdin = (
        """\
FROM langchain/langgraph-api:3.11
ADD pipconfig.txt /pipconfig.txt
# -- Adding non-package dependency unit_tests --
ADD . /deps/outer-unit_tests/unit_tests
RUN set -ex && \\
    for line in '[project]' \\
                'name = "unit_tests"' \\
                'version = "0.1"' \\
                '[tool.setuptools.package-data]' \\
                '"*" = ["**/*"]' \\
                '[build-system]' \\
                'requires = ["setuptools>=61"]' \\
                'build-backend = "setuptools.build_meta"'; do \\
        echo "$line" >> /deps/outer-unit_tests/pyproject.toml; \\
    done
# -- End of non-package dependency unit_tests --
# -- Installing all local dependencies --
RUN for dep in /deps/*; do             echo "Installing $dep";             if [ -d "$dep" ]; then                 echo "Installing $dep";                 (cd "$dep" && PIP_CONFIG_FILE=/pipconfig.txt PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt -e .);             fi;         done
# -- End of local dependencies install --
ENV LANGSERVE_GRAPHS='{"agent": "/deps/outer-unit_tests/unit_tests/agent.py:graph"}'
"""
        + FORMATTED_CLEANUP_LINES
        + """
WORKDIR /deps/outer-unit_tests/unit_tests\
"""
    )
    assert clean_empty_lines(actual_docker_stdin) == expected_docker_stdin
    assert additional_contexts == {}


def test_config_to_docker_invalid_inputs():
    # test missing local dependencies
    with pytest.raises(FileNotFoundError):
        graphs = {"agent": "tests/unit_tests/agent.py:graph"}
        config_to_docker(
            PATH_TO_CONFIG,
            validate_config({"dependencies": ["./missing"], "graphs": graphs}),
            "langchain/langgraph-api",
        )

    # test missing local module
    with pytest.raises(FileNotFoundError):
        graphs = {"agent": "./missing_agent.py:graph"}
        config_to_docker(
            PATH_TO_CONFIG,
            validate_config({"dependencies": ["."], "graphs": graphs}),
            "langchain/langgraph-api",
        )


def test_config_to_docker_local_deps():
    graphs = {"agent": "./graphs/agent.py:graph"}
    actual_docker_stdin, additional_contexts = config_to_docker(
        PATH_TO_CONFIG,
        validate_config(
            {
                "dependencies": ["./graphs"],
                "graphs": graphs,
            }
        ),
        "langchain/langgraph-api-custom",
    )
    expected_docker_stdin = f"""\
FROM langchain/langgraph-api-custom:3.11
# -- Adding non-package dependency graphs --
ADD ./graphs /deps/outer-graphs/src
RUN set -ex && \\
    for line in '[project]' \\
                'name = "graphs"' \\
                'version = "0.1"' \\
                '[tool.setuptools.package-data]' \\
                '"*" = ["**/*"]' \\
                '[build-system]' \\
                'requires = ["setuptools>=61"]' \\
                'build-backend = "setuptools.build_meta"'; do \\
        echo "$line" >> /deps/outer-graphs/pyproject.toml; \\
    done
# -- End of non-package dependency graphs --
# -- Installing all local dependencies --
RUN for dep in /deps/*; do             echo "Installing $dep";             if [ -d "$dep" ]; then                 echo "Installing $dep";                 (cd "$dep" && PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt -e .);             fi;         done
# -- End of local dependencies install --
ENV LANGSERVE_GRAPHS='{{"agent": "/deps/outer-graphs/src/agent.py:graph"}}'
{FORMATTED_CLEANUP_LINES}\
"""
    assert clean_empty_lines(actual_docker_stdin) == expected_docker_stdin
    assert additional_contexts == {}


def test_config_to_docker_pyproject():
    pyproject_str = """[project]
name = "custom"
version = "0.1"
dependencies = ["langchain"]"""
    pyproject_path = "tests/unit_tests/pyproject.toml"
    with open(pyproject_path, "w") as f:
        f.write(pyproject_str)

    graphs = {"agent": "./graphs/agent.py:graph"}
    actual_docker_stdin, additional_contexts = config_to_docker(
        PATH_TO_CONFIG,
        validate_config(
            {
                "dependencies": ["."],
                "graphs": graphs,
            }
        ),
        "langchain/langgraph-api",
    )
    os.remove(pyproject_path)
    expected_docker_stdin = (
        """FROM langchain/langgraph-api:3.11
# -- Adding local package . --
ADD . /deps/unit_tests
# -- End of local package . --
# -- Installing all local dependencies --
RUN for dep in /deps/*; do             echo "Installing $dep";             if [ -d "$dep" ]; then                 echo "Installing $dep";                 (cd "$dep" && PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt -e .);             fi;         done
# -- End of local dependencies install --
ENV LANGSERVE_GRAPHS='{"agent": "/deps/unit_tests/graphs/agent.py:graph"}'
"""
        + FORMATTED_CLEANUP_LINES
        + "\n"
        + "WORKDIR /deps/unit_tests"
        ""
    )
    assert clean_empty_lines(actual_docker_stdin) == expected_docker_stdin
    assert additional_contexts == {}


def test_config_to_docker_end_to_end():
    graphs = {"agent": "./graphs/agent.py:graph"}
    actual_docker_stdin, additional_contexts = config_to_docker(
        PATH_TO_CONFIG,
        validate_config(
            {
                "python_version": "3.12",
                "dependencies": ["./graphs/", "langchain", "langchain_openai"],
                "graphs": graphs,
                "pip_config_file": "pipconfig.txt",
                "dockerfile_lines": ["ARG meow", "ARG foo"],
            }
        ),
        "langchain/langgraph-api",
    )
    expected_docker_stdin = f"""FROM langchain/langgraph-api:3.12
ARG meow
ARG foo
ADD pipconfig.txt /pipconfig.txt
RUN PIP_CONFIG_FILE=/pipconfig.txt PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt langchain langchain_openai
# -- Adding non-package dependency graphs --
ADD ./graphs/ /deps/outer-graphs/src
RUN set -ex && \\
    for line in '[project]' \\
                'name = "graphs"' \\
                'version = "0.1"' \\
                '[tool.setuptools.package-data]' \\
                '"*" = ["**/*"]' \\
                '[build-system]' \\
                'requires = ["setuptools>=61"]' \\
                'build-backend = "setuptools.build_meta"'; do \\
        echo "$line" >> /deps/outer-graphs/pyproject.toml; \\
    done
# -- End of non-package dependency graphs --
# -- Installing all local dependencies --
RUN for dep in /deps/*; do             echo "Installing $dep";             if [ -d "$dep" ]; then                 echo "Installing $dep";                 (cd "$dep" && PIP_CONFIG_FILE=/pipconfig.txt PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt -e .);             fi;         done
# -- End of local dependencies install --
ENV LANGSERVE_GRAPHS='{{"agent": "/deps/outer-graphs/src/agent.py:graph"}}'
{FORMATTED_CLEANUP_LINES}"""
    assert clean_empty_lines(actual_docker_stdin) == expected_docker_stdin
    assert additional_contexts == {}


# node.js build used for LangSmith Deployment
def test_config_to_docker_nodejs():
    graphs = {"agent": "./graphs/agent.js:graph"}
    actual_docker_stdin, additional_contexts = config_to_docker(
        PATH_TO_CONFIG,
        validate_config(
            {
                "node_version": "20",
                "graphs": graphs,
                "dockerfile_lines": ["ARG meow", "ARG foo"],
                "auth": {"path": "./graphs/auth.mts:auth"},
                "ui": {"agent": "./graphs/agent.ui.jsx"},
                "ui_config": {"shared": ["nuqs"]},
            }
        ),
        "langchain/langgraphjs-api",
    )
    expected_docker_stdin = """FROM langchain/langgraphjs-api:20
ARG meow
ARG foo
ADD . /deps/unit_tests
RUN cd /deps/unit_tests && npm i
ENV LANGGRAPH_AUTH='{"path": "./graphs/auth.mts:auth"}'
ENV LANGGRAPH_UI='{"agent": "./graphs/agent.ui.jsx"}'
ENV LANGGRAPH_UI_CONFIG='{"shared": ["nuqs"]}'
ENV LANGSERVE_GRAPHS='{"agent": "./graphs/agent.js:graph"}'
WORKDIR /deps/unit_tests
RUN (test ! -f /api/langgraph_api/js/build.mts && echo "Prebuild script not found, skipping") || tsx /api/langgraph_api/js/build.mts"""

    assert clean_empty_lines(actual_docker_stdin) == expected_docker_stdin
    assert additional_contexts == {}


def test_config_to_docker_nodejs_internal_docker_tag():
    graphs = {"agent": "./graphs/agent.js:graph"}
    actual_docker_stdin, additional_contexts = config_to_docker(
        PATH_TO_CONFIG,
        validate_config(
            {
                "node_version": "20",
                "graphs": graphs,
                "dockerfile_lines": ["ARG meow", "ARG foo"],
                "auth": {"path": "./graphs/auth.mts:auth"},
                "ui": {"agent": "./graphs/agent.ui.jsx"},
                "ui_config": {"shared": ["nuqs"]},
                "_INTERNAL_docker_tag": "my-tag",
            }
        ),
        "langchain/langgraphjs-api",
    )
    expected_docker_stdin = """FROM langchain/langgraphjs-api:my-tag
ARG meow
ARG foo
ADD . /deps/unit_tests
RUN cd /deps/unit_tests && npm i
ENV LANGGRAPH_AUTH='{"path": "./graphs/auth.mts:auth"}'
ENV LANGGRAPH_UI='{"agent": "./graphs/agent.ui.jsx"}'
ENV LANGGRAPH_UI_CONFIG='{"shared": ["nuqs"]}'
ENV LANGSERVE_GRAPHS='{"agent": "./graphs/agent.js:graph"}'
WORKDIR /deps/unit_tests
RUN (test ! -f /api/langgraph_api/js/build.mts && echo "Prebuild script not found, skipping") || tsx /api/langgraph_api/js/build.mts"""

    assert clean_empty_lines(actual_docker_stdin) == expected_docker_stdin
    assert additional_contexts == {}


def test_config_to_docker_gen_ui_python():
    graphs = {"agent": "./agent.py:graph"}
    actual_docker_stdin, additional_contexts = config_to_docker(
        PATH_TO_CONFIG,
        validate_config(
            {
                "dependencies": ["."],
                "graphs": graphs,
                "ui": {"agent": "./graphs/agent.ui.jsx"},
                "ui_config": {"shared": ["nuqs"]},
            }
        ),
        "langchain/langgraph-api",
    )

    expected_docker_stdin = f"""FROM langchain/langgraph-api:3.11
RUN /storage/install-node.sh
# -- Adding non-package dependency unit_tests --
ADD . /deps/outer-unit_tests/unit_tests
RUN set -ex && \\
    for line in '[project]' \\
                'name = "unit_tests"' \\
                'version = "0.1"' \\
                '[tool.setuptools.package-data]' \\
                '"*" = ["**/*"]' \\
                '[build-system]' \\
                'requires = ["setuptools>=61"]' \\
                'build-backend = "setuptools.build_meta"'; do \\
        echo "$line" >> /deps/outer-unit_tests/pyproject.toml; \\
    done
# -- End of non-package dependency unit_tests --
# -- Installing all local dependencies --
RUN for dep in /deps/*; do             echo "Installing $dep";             if [ -d "$dep" ]; then                 echo "Installing $dep";                 (cd "$dep" && PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt -e .);             fi;         done
# -- End of local dependencies install --
ENV LANGGRAPH_UI='{{"agent": "./graphs/agent.ui.jsx"}}'
ENV LANGGRAPH_UI_CONFIG='{{"shared": ["nuqs"]}}'
ENV LANGSERVE_GRAPHS='{{"agent": "/deps/outer-unit_tests/unit_tests/agent.py:graph"}}'
# -- Installing JS dependencies --
ENV NODE_VERSION=20
RUN cd /deps/outer-unit_tests/unit_tests && npm i && tsx /api/langgraph_api/js/build.mts
# -- End of JS dependencies install --
{FORMATTED_CLEANUP_LINES}
WORKDIR /deps/outer-unit_tests/unit_tests"""

    assert clean_empty_lines(actual_docker_stdin) == expected_docker_stdin
    assert additional_contexts == {}


def test_config_to_docker_multiplatform():
    graphs = {
        "python": "./multiplatform/python.py:graph",
        "js": "./multiplatform/js.mts:graph",
    }
    actual_docker_stdin, additional_contexts = config_to_docker(
        PATH_TO_CONFIG,
        validate_config(
            {"node_version": "22", "dependencies": ["."], "graphs": graphs}
        ),
        "langchain/langgraph-api",
    )

    expected_docker_stdin = f"""FROM langchain/langgraph-api:3.11
RUN /storage/install-node.sh
# -- Adding non-package dependency unit_tests --
ADD . /deps/outer-unit_tests/unit_tests
RUN set -ex && \\
    for line in '[project]' \\
                'name = "unit_tests"' \\
                'version = "0.1"' \\
                '[tool.setuptools.package-data]' \\
                '"*" = ["**/*"]' \\
                '[build-system]' \\
                'requires = ["setuptools>=61"]' \\
                'build-backend = "setuptools.build_meta"'; do \\
        echo "$line" >> /deps/outer-unit_tests/pyproject.toml; \\
    done
# -- End of non-package dependency unit_tests --
# -- Installing all local dependencies --
RUN for dep in /deps/*; do             echo "Installing $dep";             if [ -d "$dep" ]; then                 echo "Installing $dep";                 (cd "$dep" && PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt -e .);             fi;         done
# -- End of local dependencies install --
ENV LANGSERVE_GRAPHS='{{"python": "/deps/outer-unit_tests/unit_tests/multiplatform/python.py:graph", "js": "/deps/outer-unit_tests/unit_tests/multiplatform/js.mts:graph"}}'
# -- Installing JS dependencies --
ENV NODE_VERSION=22
RUN cd /deps/outer-unit_tests/unit_tests && npm i && tsx /api/langgraph_api/js/build.mts
# -- End of JS dependencies install --
{FORMATTED_CLEANUP_LINES}
WORKDIR /deps/outer-unit_tests/unit_tests"""

    assert clean_empty_lines(actual_docker_stdin) == expected_docker_stdin
    assert additional_contexts == {}


def test_config_to_docker_pip_installer():
    """Test that pip_installer setting affects the generated Dockerfile."""
    graphs = {"agent": "./graphs/agent.py:graph"}
    base_config = {
        "python_version": "3.11",
        "dependencies": ["."],
        "graphs": graphs,
    }

    # Test default (auto) behavior with UV-supporting image
    config_auto = validate_config(
        {**copy.deepcopy(base_config), "pip_installer": "auto"}
    )
    docker_auto, _ = config_to_docker(
        PATH_TO_CONFIG, config_auto, "langchain/langgraph-api:0.2.47"
    )
    assert "uv pip install --system " in docker_auto
    assert "rm /usr/bin/uv /usr/bin/uvx" in docker_auto

    # Test explicit pip setting
    config_pip = validate_config({**copy.deepcopy(base_config), "pip_installer": "pip"})
    docker_pip, _ = config_to_docker(
        PATH_TO_CONFIG, config_pip, "langchain/langgraph-api:0.2.47"
    )
    assert "uv pip install --system " not in docker_pip
    assert "pip install" in docker_pip
    assert "rm /usr/bin/uv" not in docker_pip

    # Test explicit uv setting
    config_uv = validate_config({**copy.deepcopy(base_config), "pip_installer": "uv"})
    docker_uv, _ = config_to_docker(
        PATH_TO_CONFIG, config_uv, "langchain/langgraph-api:0.2.47"
    )
    assert "uv pip install --system " in docker_uv
    assert "rm /usr/bin/uv /usr/bin/uvx" in docker_uv

    # Test auto behavior with older image (should use pip)
    config_auto_old = validate_config(
        {**copy.deepcopy(base_config), "pip_installer": "auto"}
    )
    docker_auto_old, _ = config_to_docker(
        PATH_TO_CONFIG, config_auto_old, "langchain/langgraph-api:0.2.46"
    )
    assert "uv pip install --system " not in docker_auto_old
    assert "pip install" in docker_auto_old
    assert "rm /usr/bin/uv" not in docker_auto_old

    # Test that missing pip_installer defaults to auto behavior
    config_default = validate_config(copy.deepcopy(base_config))
    docker_default, _ = config_to_docker(
        PATH_TO_CONFIG, config_default, "langchain/langgraph-api:0.2.47"
    )
    assert "uv pip install --system " in docker_default


def test_config_retain_build_tools():
    graphs = {"agent": "./graphs/agent.py:graph"}
    base_config = {
        "python_version": "3.11",
        "dependencies": ["."],
        "graphs": graphs,
    }
    config_true = validate_config(
        {**copy.deepcopy(base_config), "keep_pkg_tools": True}
    )
    docker_true, _ = config_to_docker(
        PATH_TO_CONFIG, config_true, "langchain/langgraph-api:0.2.47"
    )
    assert not any(
        "/usr/local/lib/python*/site-packages/" + pckg + "*" in docker_true
        for pckg in _BUILD_TOOLS
    )
    assert "RUN pip uninstall -y pip setuptools wheel" not in docker_true
    config_false = validate_config(
        {**copy.deepcopy(base_config), "keep_pkg_tools": False}
    )
    docker_false, _ = config_to_docker(
        PATH_TO_CONFIG, config_false, "langchain/langgraph-api:0.2.47"
    )
    assert all(
        "/usr/local/lib/python*/site-packages/" + pckg + "*" in docker_false
        for pckg in _BUILD_TOOLS
    )
    assert "RUN pip uninstall -y pip setuptools wheel" in docker_false
    config_list = validate_config(
        {**copy.deepcopy(base_config), "keep_pkg_tools": ["pip", "setuptools"]}
    )
    docker_list, _ = config_to_docker(
        PATH_TO_CONFIG, config_list, "langchain/langgraph-api:0.2.47"
    )
    assert all(
        "/usr/local/lib/python*/site-packages/" + pckg + "*" in docker_list
        for pckg in ("wheel",)
    )
    assert not any(
        "/usr/local/lib/python*/site-packages/" + pckg + "*" in docker_list
        for pckg in ("pip", "setuptools")
    )
    assert "RUN pip uninstall -y wheel" in docker_list
    assert "RUN pip uninstall -y pip setuptools" not in docker_list


# config_to_compose
def test_config_to_compose_simple_config():
    graphs = {"agent": "./agent.py:graph"}
    # Create a properly indented version of FORMATTED_CLEANUP_LINES for compose files
    expected_compose_stdin = f"""
        pull_policy: build
        build:
            context: .
            dockerfile_inline: |
                FROM langchain/langgraph-api:3.11
                # -- Adding non-package dependency unit_tests --
                ADD . /deps/outer-unit_tests/unit_tests
                RUN set -ex && \\
                    for line in '[project]' \\
                                'name = "unit_tests"' \\
                                'version = "0.1"' \\
                                '[tool.setuptools.package-data]' \\
                                '"*" = ["**/*"]' \\
                                '[build-system]' \\
                                'requires = ["setuptools>=61"]' \\
                                'build-backend = "setuptools.build_meta"'; do \\
                        echo "$line" >> /deps/outer-unit_tests/pyproject.toml; \\
                    done
                # -- End of non-package dependency unit_tests --
                # -- Installing all local dependencies --
                RUN for dep in /deps/*; do             echo "Installing $dep";             if [ -d "$dep" ]; then                 echo "Installing $dep";                 (cd "$dep" && PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt -e .);             fi;         done
                # -- End of local dependencies install --
                ENV LANGSERVE_GRAPHS='{{"agent": "/deps/outer-unit_tests/unit_tests/agent.py:graph"}}'
{textwrap.indent(textwrap.dedent(FORMATTED_CLEANUP_LINES), "                ")}
                WORKDIR /deps/outer-unit_tests/unit_tests
        """
    actual_compose_stdin = config_to_compose(
        PATH_TO_CONFIG,
        validate_config({"dependencies": ["."], "graphs": graphs}),
        "langchain/langgraph-api",
    )
    assert (
        clean_empty_lines(actual_compose_stdin).strip()
        == expected_compose_stdin.strip()
    )


def test_config_to_compose_env_vars():
    graphs = {"agent": "./agent.py:graph"}
    expected_compose_stdin = f"""                        OPENAI_API_KEY: "key"
        
        pull_policy: build
        build:
            context: .
            dockerfile_inline: |
                FROM langchain/langgraph-api-custom:3.11
                # -- Adding non-package dependency unit_tests --
                ADD . /deps/outer-unit_tests/unit_tests
                RUN set -ex && \\
                    for line in '[project]' \\
                                'name = "unit_tests"' \\
                                'version = "0.1"' \\
                                '[tool.setuptools.package-data]' \\
                                '"*" = ["**/*"]' \\
                                '[build-system]' \\
                                'requires = ["setuptools>=61"]' \\
                                'build-backend = "setuptools.build_meta"'; do \\
                        echo "$line" >> /deps/outer-unit_tests/pyproject.toml; \\
                    done
                # -- End of non-package dependency unit_tests --
                # -- Installing all local dependencies --
                RUN for dep in /deps/*; do             echo "Installing $dep";             if [ -d "$dep" ]; then                 echo "Installing $dep";                 (cd "$dep" && PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt -e .);             fi;         done
                # -- End of local dependencies install --
                ENV LANGSERVE_GRAPHS='{{"agent": "/deps/outer-unit_tests/unit_tests/agent.py:graph"}}'
{textwrap.indent(textwrap.dedent(FORMATTED_CLEANUP_LINES), "                ")}
                WORKDIR /deps/outer-unit_tests/unit_tests
        """
    openai_api_key = "key"
    actual_compose_stdin = config_to_compose(
        PATH_TO_CONFIG,
        validate_config(
            {
                "dependencies": ["."],
                "graphs": graphs,
                "env": {"OPENAI_API_KEY": openai_api_key},
            }
        ),
        "langchain/langgraph-api-custom",
    )
    assert clean_empty_lines(actual_compose_stdin) == expected_compose_stdin


def test_config_to_compose_env_file():
    graphs = {"agent": "./agent.py:graph"}
    expected_compose_stdin = f"""\
        env_file: .env
        pull_policy: build
        build:
            context: .
            dockerfile_inline: |
                FROM langchain/langgraph-api:3.11
                # -- Adding non-package dependency unit_tests --
                ADD . /deps/outer-unit_tests/unit_tests
                RUN set -ex && \\
                    for line in '[project]' \\
                                'name = "unit_tests"' \\
                                'version = "0.1"' \\
                                '[tool.setuptools.package-data]' \\
                                '"*" = ["**/*"]' \\
                                '[build-system]' \\
                                'requires = ["setuptools>=61"]' \\
                                'build-backend = "setuptools.build_meta"'; do \\
                        echo "$line" >> /deps/outer-unit_tests/pyproject.toml; \\
                    done
                # -- End of non-package dependency unit_tests --
                # -- Installing all local dependencies --
                RUN for dep in /deps/*; do             echo "Installing $dep";             if [ -d "$dep" ]; then                 echo "Installing $dep";                 (cd "$dep" && PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt -e .);             fi;         done
                # -- End of local dependencies install --
                ENV LANGSERVE_GRAPHS='{{"agent": "/deps/outer-unit_tests/unit_tests/agent.py:graph"}}'
{textwrap.indent(textwrap.dedent(FORMATTED_CLEANUP_LINES), "                ")}
                WORKDIR /deps/outer-unit_tests/unit_tests
        """
    actual_compose_stdin = config_to_compose(
        PATH_TO_CONFIG,
        validate_config({"dependencies": ["."], "graphs": graphs, "env": ".env"}),
        "langchain/langgraph-api",
    )
    assert clean_empty_lines(actual_compose_stdin) == expected_compose_stdin


def test_config_to_compose_watch():
    graphs = {"agent": "./agent.py:graph"}
    expected_compose_stdin = f"""\
        
        pull_policy: build
        build:
            context: .
            dockerfile_inline: |
                FROM langchain/langgraph-api:3.11
                # -- Adding non-package dependency unit_tests --
                ADD . /deps/outer-unit_tests/unit_tests
                RUN set -ex && \\
                    for line in '[project]' \\
                                'name = "unit_tests"' \\
                                'version = "0.1"' \\
                                '[tool.setuptools.package-data]' \\
                                '"*" = ["**/*"]' \\
                                '[build-system]' \\
                                'requires = ["setuptools>=61"]' \\
                                'build-backend = "setuptools.build_meta"'; do \\
                        echo "$line" >> /deps/outer-unit_tests/pyproject.toml; \\
                    done
                # -- End of non-package dependency unit_tests --
                # -- Installing all local dependencies --
                RUN for dep in /deps/*; do             echo "Installing $dep";             if [ -d "$dep" ]; then                 echo "Installing $dep";                 (cd "$dep" && PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt -e .);             fi;         done
                # -- End of local dependencies install --
                ENV LANGSERVE_GRAPHS='{{"agent": "/deps/outer-unit_tests/unit_tests/agent.py:graph"}}'
{textwrap.indent(textwrap.dedent(FORMATTED_CLEANUP_LINES), "                ")}
                WORKDIR /deps/outer-unit_tests/unit_tests
        
        develop:
            watch:
                - path: test_config.json
                  action: rebuild
                - path: .
                  action: rebuild\
"""
    actual_compose_stdin = config_to_compose(
        PATH_TO_CONFIG,
        validate_config({"dependencies": ["."], "graphs": graphs}),
        "langchain/langgraph-api",
        watch=True,
    )
    assert clean_empty_lines(actual_compose_stdin) == expected_compose_stdin


def test_config_to_compose_end_to_end():
    # test all of the above + langgraph API path
    graphs = {"agent": "./agent.py:graph"}
    expected_compose_stdin = f"""\
        env_file: .env
        pull_policy: build
        build:
            context: .
            dockerfile_inline: |
                FROM langchain/langgraph-api:3.11
                # -- Adding non-package dependency unit_tests --
                ADD . /deps/outer-unit_tests/unit_tests
                RUN set -ex && \\
                    for line in '[project]' \\
                                'name = "unit_tests"' \\
                                'version = "0.1"' \\
                                '[tool.setuptools.package-data]' \\
                                '"*" = ["**/*"]' \\
                                '[build-system]' \\
                                'requires = ["setuptools>=61"]' \\
                                'build-backend = "setuptools.build_meta"'; do \\
                        echo "$line" >> /deps/outer-unit_tests/pyproject.toml; \\
                    done
                # -- End of non-package dependency unit_tests --
                # -- Installing all local dependencies --
                RUN for dep in /deps/*; do             echo "Installing $dep";             if [ -d "$dep" ]; then                 echo "Installing $dep";                 (cd "$dep" && PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt -e .);             fi;         done
                # -- End of local dependencies install --
                ENV LANGSERVE_GRAPHS='{{"agent": "/deps/outer-unit_tests/unit_tests/agent.py:graph"}}'
{textwrap.indent(textwrap.dedent(FORMATTED_CLEANUP_LINES), "                ")}
                WORKDIR /deps/outer-unit_tests/unit_tests
        
        develop:
            watch:
                - path: test_config.json
                  action: rebuild
                - path: .
                  action: rebuild\
"""
    actual_compose_stdin = config_to_compose(
        PATH_TO_CONFIG,
        validate_config({"dependencies": ["."], "graphs": graphs, "env": ".env"}),
        "langchain/langgraph-api",
        watch=True,
    )
    assert clean_empty_lines(actual_compose_stdin) == expected_compose_stdin


def test_docker_tag_image_distro():
    """Test docker_tag function with different image_distro configurations."""

    # Test 1: Default distro (debian) - no suffix
    config = validate_config(
        {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
        }
    )
    tag = docker_tag(config)
    assert tag == "langchain/langgraph-api:3.11"

    # Test 2: Explicit debian distro - no suffix (same as default)
    config = validate_config(
        {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
            "image_distro": "debian",
        }
    )
    tag = docker_tag(config)
    assert tag == "langchain/langgraph-api:3.11"

    # Test 3: Wolfi distro - should add suffix
    config = validate_config(
        {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
            "image_distro": "wolfi",
        }
    )
    tag = docker_tag(config)
    assert tag == "langchain/langgraph-api:3.11-wolfi"

    # Test 4: Node.js with default distro
    config = validate_config(
        {
            "node_version": "20",
            "graphs": {"agent": "./agent.js:graph"},
        }
    )
    tag = docker_tag(config)
    assert tag == "langchain/langgraphjs-api:20"

    # Test 5: Node.js with wolfi distro
    config = validate_config(
        {
            "node_version": "20",
            "graphs": {"agent": "./agent.js:graph"},
            "image_distro": "wolfi",
        }
    )
    tag = docker_tag(config)
    assert tag == "langchain/langgraphjs-api:20-wolfi"

    # Test 6: Custom base image with wolfi
    config = validate_config(
        {
            "python_version": "3.12",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
            "image_distro": "wolfi",
            "base_image": "my-registry/custom-image",
        }
    )
    tag = docker_tag(config, base_image="my-registry/custom-image")
    assert tag == "my-registry/custom-image:3.12-wolfi"


def test_docker_tag_multiplatform_with_distro():
    """Test docker_tag with multiplatform configs and image_distro."""

    # Test 1: Multiplatform (Python + Node) with wolfi
    config = validate_config(
        {
            "python_version": "3.11",
            "node_version": "20",
            "dependencies": ["."],
            "graphs": {"python": "./agent.py:graph", "js": "./agent.js:graph"},
            "image_distro": "wolfi",
        }
    )
    tag = docker_tag(config)
    # Should default to Python when both are present
    assert tag == "langchain/langgraph-api:3.11-wolfi"

    # Test 2: Node-only multiplatform with wolfi
    config = validate_config(
        {
            "node_version": "20",
            "graphs": {"js": "./agent.js:graph"},
            "image_distro": "wolfi",
        }
    )
    tag = docker_tag(config)
    assert tag == "langchain/langgraphjs-api:20-wolfi"


def test_docker_tag_different_python_versions_with_distro():
    """Test docker_tag with different Python versions and distros."""

    versions_and_expected = [
        ("3.11", "langchain/langgraph-api:3.11-wolfi"),
        ("3.12", "langchain/langgraph-api:3.12-wolfi"),
        ("3.13", "langchain/langgraph-api:3.13-wolfi"),
    ]

    for python_version, expected_tag in versions_and_expected:
        config = validate_config(
            {
                "python_version": python_version,
                "dependencies": ["."],
                "graphs": {"agent": "./agent.py:graph"},
                "image_distro": "wolfi",
            }
        )
        tag = docker_tag(config)
        assert tag == expected_tag, f"Failed for Python {python_version}"


def test_docker_tag_different_node_versions_with_distro():
    """Test docker_tag with different Node.js versions and distros."""

    versions_and_expected = [
        ("20", "langchain/langgraphjs-api:20-wolfi"),
        ("21", "langchain/langgraphjs-api:21-wolfi"),
        ("22", "langchain/langgraphjs-api:22-wolfi"),
    ]

    for node_version, expected_tag in versions_and_expected:
        config = validate_config(
            {
                "node_version": node_version,
                "graphs": {"agent": "./agent.js:graph"},
                "image_distro": "wolfi",
            }
        )
        tag = docker_tag(config)
        assert tag == expected_tag, f"Failed for Node.js {node_version}"


@pytest.mark.parametrize("in_config", [False, True])
def test_docker_tag_with_api_version(in_config: bool):
    """Test docker_tag function with api_version parameter."""

    # Test 1: Python config with api_version and default distro
    version = "0.2.74"
    config = validate_config(
        {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
            "api_version": version if in_config else None,
        }
    )
    tag = docker_tag(config, api_version=version if not in_config else None)
    assert tag == f"langchain/langgraph-api:{version}-py3.11"

    # Test 2: Python config with api_version and wolfi distro
    config = validate_config(
        {
            "python_version": "3.12",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
            "image_distro": "wolfi",
            "api_version": version if in_config else None,
        }
    )
    tag = docker_tag(config, api_version=version if not in_config else None)
    assert tag == f"langchain/langgraph-api:{version}-py3.12-wolfi"

    # Test 3: Node.js config with api_version and default distro
    config = validate_config(
        {
            "node_version": "20",
            "graphs": {"agent": "./agent.js:graph"},
            "api_version": version if in_config else None,
        }
    )
    tag = docker_tag(config, api_version=version if not in_config else None)
    assert tag == f"langchain/langgraphjs-api:{version}-node20"

    # Test 4: Node.js config with api_version and wolfi distro
    config = validate_config(
        {
            "node_version": "20",
            "graphs": {"agent": "./agent.js:graph"},
            "image_distro": "wolfi",
            "api_version": version if in_config else None,
        }
    )
    tag = docker_tag(config, api_version=version if not in_config else None)
    assert tag == f"langchain/langgraphjs-api:{version}-node20-wolfi"

    # Test 5: Custom base image with api_version
    config = validate_config(
        {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
            "base_image": "my-registry/custom-image",
            "api_version": version if in_config else None,
        }
    )
    tag = docker_tag(
        config,
        base_image="my-registry/custom-image",
        api_version=version if not in_config else None,
    )
    assert tag == f"my-registry/custom-image:{version}-py3.11"

    # Test 6: api_version with different Python versions
    for python_version in ["3.11", "3.12", "3.13"]:
        config = validate_config(
            {
                "python_version": python_version,
                "dependencies": ["."],
                "graphs": {"agent": "./agent.py:graph"},
                "api_version": version if in_config else None,
            }
        )
        tag = docker_tag(config, api_version=version if not in_config else None)
        assert tag == f"langchain/langgraph-api:{version}-py{python_version}"

    # Test 7: Without api_version should work as before
    config = validate_config(
        {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
        }
    )
    tag = docker_tag(config)
    assert tag == "langchain/langgraph-api:3.11"

    # Test 8: api_version with multiplatform config (should default to Python)
    config = validate_config(
        {
            "python_version": "3.11",
            "node_version": "20",
            "dependencies": ["."],
            "graphs": {"python": "./agent.py:graph", "js": "./agent.js:graph"},
            "api_version": version if in_config else None,
        }
    )
    tag = docker_tag(config, api_version=version if not in_config else None)
    assert tag == f"langchain/langgraph-api:{version}-py3.11"

    # Test 9: api_version with _INTERNAL_docker_tag should ignore api_version
    config = validate_config(
        {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
            "_INTERNAL_docker_tag": "internal-tag",
        }
    )
    tag = docker_tag(config, api_version="0.2.74")
    assert tag == "langchain/langgraph-api:internal-tag"

    # Test 10: api_version with langgraph-server base image should follow special format
    config = validate_config(
        {
            "python_version": "3.11",
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
            "api_version": version if in_config else None,
        }
    )
    tag = docker_tag(
        config,
        base_image="langchain/langgraph-server",
        api_version=version if not in_config else None,
    )
    assert tag == f"langchain/langgraph-server:{version}-py3.11"


def test_config_to_docker_with_api_version():
    """Test config_to_docker function with api_version parameter."""

    # Test Python config with api_version
    graphs = {"agent": "./agent.py:graph"}
    actual_docker_stdin, additional_contexts = config_to_docker(
        PATH_TO_CONFIG,
        validate_config({"dependencies": ["."], "graphs": graphs}),
        "langchain/langgraph-api",
        api_version="0.2.74",
    )

    # Check that the FROM line uses the api_version
    lines = actual_docker_stdin.split("\n")
    from_line = lines[0]
    assert from_line == "FROM langchain/langgraph-api:0.2.74-py3.11"

    # Test Node.js config with api_version
    graphs = {"agent": "./agent.js:graph"}
    actual_docker_stdin, additional_contexts = config_to_docker(
        PATH_TO_CONFIG,
        validate_config({"node_version": "20", "graphs": graphs}),
        "langchain/langgraphjs-api",
        api_version="0.2.74",
    )

    # Check that the FROM line uses the api_version
    lines = actual_docker_stdin.split("\n")
    from_line = lines[0]
    assert from_line == "FROM langchain/langgraphjs-api:0.2.74-node20"


def test_config_to_compose_with_api_version():
    """Test config_to_compose function with api_version parameter."""

    # Test Python config with api_version
    config = validate_config(
        {
            "dependencies": ["."],
            "graphs": {"agent": "./agent.py:graph"},
        }
    )

    actual_compose_str = config_to_compose(
        PATH_TO_CONFIG,
        config,
        "langchain/langgraph-api",
        api_version="0.2.74",
    )

    # Check that the compose file includes the correct FROM line with api_version
    assert "FROM langchain/langgraph-api:0.2.74-py3.11" in actual_compose_str

    # Test Node.js config with api_version
    config = validate_config(
        {
            "node_version": "20",
            "graphs": {"agent": "./agent.js:graph"},
        }
    )

    actual_compose_str = config_to_compose(
        PATH_TO_CONFIG,
        config,
        "langchain/langgraphjs-api",
        api_version="0.2.74",
    )

    # Check that the compose file includes the correct FROM line with api_version
    assert "FROM langchain/langgraphjs-api:0.2.74-node20" in actual_compose_str

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/test_docker.py
```py
import pytest

from langgraph_cli.docker import (
    DEFAULT_POSTGRES_URI,
    DockerCapabilities,
    Version,
    _parse_version,
    compose,
)
from langgraph_cli.util import clean_empty_lines

DEFAULT_DOCKER_CAPABILITIES = DockerCapabilities(
    version_docker=Version(26, 1, 1),
    version_compose=Version(2, 27, 0),
    healthcheck_start_interval=False,
)


def test_compose_with_no_debugger_and_custom_db():
    port = 8123
    custom_postgres_uri = "custom_postgres_uri"
    actual_compose_str = compose(
        DEFAULT_DOCKER_CAPABILITIES, port=port, postgres_uri=custom_postgres_uri
    )
    expected_compose_str = f"""services:
    langgraph-redis:
        image: redis:6
        healthcheck:
            test: redis-cli ping
            interval: 5s
            timeout: 1s
            retries: 5
    langgraph-api:
        ports:
            - "{port}:8000"
        depends_on:
            langgraph-redis:
                condition: service_healthy
        environment:
            REDIS_URI: redis://langgraph-redis:6379
            POSTGRES_URI: {custom_postgres_uri}"""
    assert clean_empty_lines(actual_compose_str) == expected_compose_str


def test_compose_with_no_debugger_and_custom_db_with_healthcheck():
    port = 8123
    custom_postgres_uri = "custom_postgres_uri"
    actual_compose_str = compose(
        DEFAULT_DOCKER_CAPABILITIES._replace(healthcheck_start_interval=True),
        port=port,
        postgres_uri=custom_postgres_uri,
    )
    expected_compose_str = f"""services:
    langgraph-redis:
        image: redis:6
        healthcheck:
            test: redis-cli ping
            interval: 5s
            timeout: 1s
            retries: 5
    langgraph-api:
        ports:
            - "{port}:8000"
        depends_on:
            langgraph-redis:
                condition: service_healthy
        environment:
            REDIS_URI: redis://langgraph-redis:6379
            POSTGRES_URI: {custom_postgres_uri}
        healthcheck:
            test: python /api/healthcheck.py
            interval: 60s
            start_interval: 1s
            start_period: 10s"""
    assert clean_empty_lines(actual_compose_str) == expected_compose_str


def test_compose_with_debugger_and_custom_db():
    port = 8123
    custom_postgres_uri = "custom_postgres_uri"
    actual_compose_str = compose(
        DEFAULT_DOCKER_CAPABILITIES,
        port=port,
        postgres_uri=custom_postgres_uri,
    )
    expected_compose_str = f"""services:
    langgraph-redis:
        image: redis:6
        healthcheck:
            test: redis-cli ping
            interval: 5s
            timeout: 1s
            retries: 5
    langgraph-api:
        ports:
            - "{port}:8000"
        depends_on:
            langgraph-redis:
                condition: service_healthy
        environment:
            REDIS_URI: redis://langgraph-redis:6379
            POSTGRES_URI: {custom_postgres_uri}"""
    assert clean_empty_lines(actual_compose_str) == expected_compose_str


def test_compose_with_debugger_and_default_db():
    port = 8123
    actual_compose_str = compose(DEFAULT_DOCKER_CAPABILITIES, port=port)
    expected_compose_str = f"""volumes:
    langgraph-data:
        driver: local
services:
    langgraph-redis:
        image: redis:6
        healthcheck:
            test: redis-cli ping
            interval: 5s
            timeout: 1s
            retries: 5
    langgraph-postgres:
        image: pgvector/pgvector:pg16
        ports:
            - "5433:5432"
        environment:
            POSTGRES_DB: postgres
            POSTGRES_USER: postgres
            POSTGRES_PASSWORD: postgres
        command:
            - postgres
            - -c
            - shared_preload_libraries=vector
        volumes:
            - langgraph-data:/var/lib/postgresql/data
        healthcheck:
            test: pg_isready -U postgres
            start_period: 10s
            timeout: 1s
            retries: 5
            interval: 5s
    langgraph-api:
        ports:
            - "{port}:8000"
        depends_on:
            langgraph-redis:
                condition: service_healthy
            langgraph-postgres:
                condition: service_healthy
        environment:
            REDIS_URI: redis://langgraph-redis:6379
            POSTGRES_URI: {DEFAULT_POSTGRES_URI}"""
    assert clean_empty_lines(actual_compose_str) == expected_compose_str


def test_compose_with_api_version():
    """Test compose function with api_version parameter."""
    port = 8123
    api_version = "0.2.74"

    actual_compose_str = compose(
        DEFAULT_DOCKER_CAPABILITIES, port=port, api_version=api_version
    )

    # The compose function should generate a compose file that doesn't directly
    # reference the api_version, since it's handled in the docker tag creation
    # when building the image. The compose function mainly sets up services.
    expected_compose_str = f"""volumes:
    langgraph-data:
        driver: local
services:
    langgraph-redis:
        image: redis:6
        healthcheck:
            test: redis-cli ping
            interval: 5s
            timeout: 1s
            retries: 5
    langgraph-postgres:
        image: pgvector/pgvector:pg16
        ports:
            - "5433:5432"
        environment:
            POSTGRES_DB: postgres
            POSTGRES_USER: postgres
            POSTGRES_PASSWORD: postgres
        command:
            - postgres
            - -c
            - shared_preload_libraries=vector
        volumes:
            - langgraph-data:/var/lib/postgresql/data
        healthcheck:
            test: pg_isready -U postgres
            start_period: 10s
            timeout: 1s
            retries: 5
            interval: 5s
    langgraph-api:
        ports:
            - "{port}:8000"
        depends_on:
            langgraph-redis:
                condition: service_healthy
            langgraph-postgres:
                condition: service_healthy
        environment:
            REDIS_URI: redis://langgraph-redis:6379
            POSTGRES_URI: {DEFAULT_POSTGRES_URI}"""
    assert clean_empty_lines(actual_compose_str) == expected_compose_str


def test_compose_with_api_version_and_base_image():
    """Test compose function with both api_version and base_image parameters."""
    port = 8123
    api_version = "1.0.0"
    base_image = "my-registry/custom-api"

    actual_compose_str = compose(
        DEFAULT_DOCKER_CAPABILITIES,
        port=port,
        api_version=api_version,
        base_image=base_image,
    )

    # Similar to the previous test - the compose function doesn't directly embed
    # the api_version or base_image into the compose file since those are handled
    # during the docker build process
    expected_compose_str = f"""volumes:
    langgraph-data:
        driver: local
services:
    langgraph-redis:
        image: redis:6
        healthcheck:
            test: redis-cli ping
            interval: 5s
            timeout: 1s
            retries: 5
    langgraph-postgres:
        image: pgvector/pgvector:pg16
        ports:
            - "5433:5432"
        environment:
            POSTGRES_DB: postgres
            POSTGRES_USER: postgres
            POSTGRES_PASSWORD: postgres
        command:
            - postgres
            - -c
            - shared_preload_libraries=vector
        volumes:
            - langgraph-data:/var/lib/postgresql/data
        healthcheck:
            test: pg_isready -U postgres
            start_period: 10s
            timeout: 1s
            retries: 5
            interval: 5s
    langgraph-api:
        ports:
            - "{port}:8000"
        depends_on:
            langgraph-redis:
                condition: service_healthy
            langgraph-postgres:
                condition: service_healthy
        environment:
            REDIS_URI: redis://langgraph-redis:6379
            POSTGRES_URI: {DEFAULT_POSTGRES_URI}"""
    assert clean_empty_lines(actual_compose_str) == expected_compose_str


def test_compose_with_api_version_and_custom_postgres():
    """Test compose function with api_version and custom postgres URI."""
    port = 8123
    api_version = "0.2.74"
    custom_postgres_uri = "postgresql://user:pass@external-db:5432/mydb"

    actual_compose_str = compose(
        DEFAULT_DOCKER_CAPABILITIES,
        port=port,
        api_version=api_version,
        postgres_uri=custom_postgres_uri,
    )

    expected_compose_str = f"""services:
    langgraph-redis:
        image: redis:6
        healthcheck:
            test: redis-cli ping
            interval: 5s
            timeout: 1s
            retries: 5
    langgraph-api:
        ports:
            - "{port}:8000"
        depends_on:
            langgraph-redis:
                condition: service_healthy
        environment:
            REDIS_URI: redis://langgraph-redis:6379
            POSTGRES_URI: {custom_postgres_uri}"""
    assert clean_empty_lines(actual_compose_str) == expected_compose_str


def test_compose_with_api_version_and_debugger():
    """Test compose function with api_version and debugger port."""
    port = 8123
    debugger_port = 8001
    api_version = "0.2.74"

    actual_compose_str = compose(
        DEFAULT_DOCKER_CAPABILITIES,
        port=port,
        api_version=api_version,
        debugger_port=debugger_port,
    )

    expected_compose_str = f"""volumes:
    langgraph-data:
        driver: local
services:
    langgraph-redis:
        image: redis:6
        healthcheck:
            test: redis-cli ping
            interval: 5s
            timeout: 1s
            retries: 5
    langgraph-postgres:
        image: pgvector/pgvector:pg16
        ports:
            - "5433:5432"
        environment:
            POSTGRES_DB: postgres
            POSTGRES_USER: postgres
            POSTGRES_PASSWORD: postgres
        command:
            - postgres
            - -c
            - shared_preload_libraries=vector
        volumes:
            - langgraph-data:/var/lib/postgresql/data
        healthcheck:
            test: pg_isready -U postgres
            start_period: 10s
            timeout: 1s
            retries: 5
            interval: 5s
    langgraph-debugger:
        image: langchain/langgraph-debugger
        restart: on-failure
        depends_on:
            langgraph-postgres:
                condition: service_healthy
        ports:
            - "{debugger_port}:3968"
    langgraph-api:
        ports:
            - "{port}:8000"
        depends_on:
            langgraph-redis:
                condition: service_healthy
            langgraph-postgres:
                condition: service_healthy
        environment:
            REDIS_URI: redis://langgraph-redis:6379
            POSTGRES_URI: {DEFAULT_POSTGRES_URI}"""
    assert clean_empty_lines(actual_compose_str) == expected_compose_str


@pytest.mark.parametrize(
    "input_str,expected",
    [
        ("1.2.3", Version(1, 2, 3)),
        ("v1.2.3", Version(1, 2, 3)),
        ("1.2.3-alpha", Version(1, 2, 3)),
        ("1.2.3+1", Version(1, 2, 3)),
        ("1.2.3-alpha+build", Version(1, 2, 3)),
        ("1.2", Version(1, 2, 0)),
        ("1", Version(1, 0, 0)),
        ("v28.1.1+1", Version(28, 1, 1)),
        ("2.0.0-beta.1+exp.sha.5114f85", Version(2, 0, 0)),
        ("v3.4.5-rc1+build.123", Version(3, 4, 5)),
    ],
)
def test_parse_version_w_edge_cases(input_str, expected):
    assert _parse_version(input_str) == expected

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/test_util.py
```py
from unittest.mock import patch

from langgraph_cli.util import clean_empty_lines, warn_non_wolfi_distro


def test_clean_empty_lines():
    """Test clean_empty_lines function."""
    # Test with empty lines
    input_str = "line1\n\nline2\n\nline3"
    result = clean_empty_lines(input_str)
    assert result == "line1\nline2\nline3"

    # Test with no empty lines
    input_str = "line1\nline2\nline3"
    result = clean_empty_lines(input_str)
    assert result == "line1\nline2\nline3"

    # Test with only empty lines
    input_str = "\n\n\n"
    result = clean_empty_lines(input_str)
    assert result == ""

    # Test empty string
    input_str = ""
    result = clean_empty_lines(input_str)
    assert result == ""


def test_warn_non_wolfi_distro_with_debian(capsys):
    """Test that warning is shown when image_distro is 'debian'."""
    config = {"image_distro": "debian"}

    warn_non_wolfi_distro(config)

    captured = capsys.readouterr()
    assert (
        "⚠️  Security Recommendation: Consider switching to Wolfi Linux for enhanced security."
        in captured.out
    )
    assert (
        "Wolfi is a security-oriented, minimal Linux distribution designed for containers."
        in captured.out
    )
    assert (
        'To switch, add \'"image_distro": "wolfi"\' to your langgraph.json config file.'
        in captured.out
    )


def test_warn_non_wolfi_distro_with_default_debian(capsys):
    """Test that warning is shown when image_distro is missing (defaults to debian)."""
    config = {}  # No image_distro key, should default to debian

    warn_non_wolfi_distro(config)

    captured = capsys.readouterr()
    assert (
        "⚠️  Security Recommendation: Consider switching to Wolfi Linux for enhanced security."
        in captured.out
    )
    assert (
        "Wolfi is a security-oriented, minimal Linux distribution designed for containers."
        in captured.out
    )
    assert (
        'To switch, add \'"image_distro": "wolfi"\' to your langgraph.json config file.'
        in captured.out
    )


def test_warn_non_wolfi_distro_with_wolfi(capsys):
    """Test that no warning is shown when image_distro is 'wolfi'."""
    config = {"image_distro": "wolfi"}

    warn_non_wolfi_distro(config)

    captured = capsys.readouterr()
    assert captured.out == ""  # No output should be generated


def test_warn_non_wolfi_distro_with_other_distro(capsys):
    """Test that warning is shown when image_distro is something other than 'wolfi'."""
    config = {"image_distro": "ubuntu"}

    warn_non_wolfi_distro(config)

    captured = capsys.readouterr()
    assert (
        "⚠️  Security Recommendation: Consider switching to Wolfi Linux for enhanced security."
        in captured.out
    )
    assert (
        "Wolfi is a security-oriented, minimal Linux distribution designed for containers."
        in captured.out
    )
    assert (
        'To switch, add \'"image_distro": "wolfi"\' to your langgraph.json config file.'
        in captured.out
    )


def test_warn_non_wolfi_distro_output_formatting():
    """Test that the warning output is properly formatted with colors and empty line."""
    config = {"image_distro": "debian"}

    with patch("click.secho") as mock_secho:
        warn_non_wolfi_distro(config)

    # Verify click.secho was called with the correct parameters
    expected_calls = [
        (
            (
                "⚠️  Security Recommendation: Consider switching to Wolfi Linux for enhanced security.",
            ),
            {"fg": "yellow", "bold": True},
        ),
        (
            (
                "   Wolfi is a security-oriented, minimal Linux distribution designed for containers.",
            ),
            {"fg": "yellow"},
        ),
        (
            (
                '   To switch, add \'"image_distro": "wolfi"\' to your langgraph.json config file.',
            ),
            {"fg": "yellow"},
        ),
        (
            ("",),  # Empty line
            {},
        ),
    ]

    assert mock_secho.call_count == 4
    for i, (expected_args, expected_kwargs) in enumerate(expected_calls):
        actual_call = mock_secho.call_args_list[i]
        assert actual_call.args == expected_args
        assert actual_call.kwargs == expected_kwargs


def test_warn_non_wolfi_distro_various_configs(capsys):
    """Test warn_non_wolfi_distro with various config scenarios."""
    test_cases = [
        # (config, should_warn, description)
        ({"image_distro": "debian"}, True, "explicit debian"),
        ({"image_distro": "wolfi"}, False, "explicit wolfi"),
        ({}, True, "missing image_distro (defaults to debian)"),
        ({"image_distro": "alpine"}, True, "other distro"),
        ({"image_distro": "ubuntu"}, True, "ubuntu distro"),
        ({"other_config": "value"}, True, "unrelated config keys"),
    ]

    for config, should_warn, description in test_cases:
        # Clear any previous output
        capsys.readouterr()

        warn_non_wolfi_distro(config)

        captured = capsys.readouterr()
        if should_warn:
            assert "⚠️  Security Recommendation" in captured.out, (
                f"Should warn for {description}"
            )
            assert "Wolfi" in captured.out, f"Should mention Wolfi for {description}"
        else:
            assert captured.out == "", f"Should not warn for {description}"


def test_warn_non_wolfi_distro_return_value():
    """Test that warn_non_wolfi_distro returns None."""
    config = {"image_distro": "debian"}
    result = warn_non_wolfi_distro(config)
    assert result is None

    config = {"image_distro": "wolfi"}
    result = warn_non_wolfi_distro(config)
    assert result is None


def test_warn_non_wolfi_distro_does_not_modify_config():
    """Test that warn_non_wolfi_distro does not modify the input config."""
    original_config = {"image_distro": "debian", "other_key": "value"}
    config_copy = original_config.copy()

    warn_non_wolfi_distro(config_copy)

    assert config_copy == original_config  # Config should remain unchanged

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/multiplatform/js.mts
```mts

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/multiplatform/python.py
```py

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/cli/__init__.py
```py

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/cli/langgraph.json
```json

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/cli/pyproject.toml
```toml

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/cli/test_cli.py
```py
import json
import pathlib
import re
import shutil
import tempfile
import textwrap
from contextlib import contextmanager
from pathlib import Path

from click.testing import CliRunner

from langgraph_cli.cli import cli, prepare_args_and_stdin
from langgraph_cli.config import Config, _get_pip_cleanup_lines, validate_config
from langgraph_cli.docker import DEFAULT_POSTGRES_URI, DockerCapabilities, Version
from langgraph_cli.util import clean_empty_lines

FORMATTED_CLEANUP_LINES = _get_pip_cleanup_lines(
    install_cmd="uv pip install --system",
    to_uninstall=("pip", "setuptools", "wheel"),
    pip_installer="uv",
)
DEFAULT_DOCKER_CAPABILITIES = DockerCapabilities(
    version_docker=Version(26, 1, 1),
    version_compose=Version(2, 27, 0),
    healthcheck_start_interval=True,
)


@contextmanager
def temporary_config_folder(config_content: dict, levels: int = 0):
    # Create a temporary directory
    temp_dir = tempfile.mkdtemp()
    try:
        # Define the path for the config.json file
        config_path = Path(temp_dir) / f"{'a/' * levels}config.json"
        # Ensure the parent directory exists
        config_path.parent.mkdir(parents=True, exist_ok=True)

        # Write the provided dictionary content to config.json
        with open(config_path, "w", encoding="utf-8") as config_file:
            json.dump(config_content, config_file)

        # Yield the temporary directory path for use within the context
        yield config_path.parent
    finally:
        # Cleanup the temporary directory and its contents
        shutil.rmtree(temp_dir)


def test_prepare_args_and_stdin() -> None:
    # this basically serves as an end-to-end test for using config and docker helpers
    config_path = pathlib.Path(__file__).parent / "langgraph.json"
    config = validate_config(
        Config(dependencies=[".", "../../.."], graphs={"agent": "agent.py:graph"})
    )
    port = 8000
    debugger_port = 8001
    debugger_graph_url = f"http://127.0.0.1:{port}"

    actual_args, actual_stdin = prepare_args_and_stdin(
        capabilities=DEFAULT_DOCKER_CAPABILITIES,
        config_path=config_path,
        config=config,
        docker_compose=pathlib.Path("custom-docker-compose.yml"),
        port=port,
        debugger_port=debugger_port,
        debugger_base_url=debugger_graph_url,
        watch=True,
    )

    expected_args = [
        "--project-directory",
        str(pathlib.Path(__file__).parent.absolute()),
        "-f",
        "custom-docker-compose.yml",
        "-f",
        "-",
    ]
    expected_stdin = f"""volumes:
    langgraph-data:
        driver: local
services:
    langgraph-redis:
        image: redis:6
        healthcheck:
            test: redis-cli ping
            interval: 5s
            timeout: 1s
            retries: 5
    langgraph-postgres:
        image: pgvector/pgvector:pg16
        ports:
            - "5433:5432"
        environment:
            POSTGRES_DB: postgres
            POSTGRES_USER: postgres
            POSTGRES_PASSWORD: postgres
        command:
            - postgres
            - -c
            - shared_preload_libraries=vector
        volumes:
            - langgraph-data:/var/lib/postgresql/data
        healthcheck:
            test: pg_isready -U postgres
            start_period: 10s
            timeout: 1s
            retries: 5
            interval: 60s
            start_interval: 1s
    langgraph-debugger:
        image: langchain/langgraph-debugger
        restart: on-failure
        depends_on:
            langgraph-postgres:
                condition: service_healthy
        ports:
            - "{debugger_port}:3968"
        environment:
            VITE_STUDIO_LOCAL_GRAPH_URL: {debugger_graph_url}
    langgraph-api:
        ports:
            - "8000:8000"
        depends_on:
            langgraph-redis:
                condition: service_healthy
            langgraph-postgres:
                condition: service_healthy
        environment:
            REDIS_URI: redis://langgraph-redis:6379
            POSTGRES_URI: {DEFAULT_POSTGRES_URI}
        healthcheck:
            test: python /api/healthcheck.py
            interval: 60s
            start_interval: 1s
            start_period: 10s
        
        pull_policy: build
        build:
            context: .
            additional_contexts:
                - cli_1: {str(pathlib.Path(__file__).parent.parent.parent.parent.absolute())}
            dockerfile_inline: |
                # syntax=docker/dockerfile:1.4
                FROM langchain/langgraph-api:3.11
                # -- Adding local package . --
                ADD . /deps/cli
                # -- End of local package . --
                # -- Adding local package ../../.. --
                COPY --from=cli_1 . /deps/cli_1
                # -- End of local package ../../.. --
                # -- Installing all local dependencies --
                RUN for dep in /deps/*; do             echo "Installing $dep";             if [ -d "$dep" ]; then                 echo "Installing $dep";                 (cd "$dep" && PYTHONDONTWRITEBYTECODE=1 uv pip install --system --no-cache-dir -c /api/constraints.txt -e .);             fi;         done
                # -- End of local dependencies install --
                ENV LANGSERVE_GRAPHS='{{"agent": "agent.py:graph"}}'
{textwrap.indent(textwrap.dedent(FORMATTED_CLEANUP_LINES), "                ")}
                WORKDIR /deps/cli
        
        develop:
            watch:
                - path: langgraph.json
                  action: rebuild
                - path: .
                  action: rebuild
                - path: ../../..
                  action: rebuild\
"""
    assert actual_args == expected_args
    assert clean_empty_lines(actual_stdin) == expected_stdin


def test_prepare_args_and_stdin_with_image() -> None:
    # this basically serves as an end-to-end test for using config and docker helpers
    config_path = pathlib.Path(__file__).parent / "langgraph.json"
    config = validate_config(
        Config(dependencies=[".", "../../.."], graphs={"agent": "agent.py:graph"})
    )
    port = 8000
    debugger_port = 8001
    debugger_graph_url = f"http://127.0.0.1:{port}"

    actual_args, actual_stdin = prepare_args_and_stdin(
        capabilities=DEFAULT_DOCKER_CAPABILITIES,
        config_path=config_path,
        config=config,
        docker_compose=pathlib.Path("custom-docker-compose.yml"),
        port=port,
        debugger_port=debugger_port,
        debugger_base_url=debugger_graph_url,
        watch=True,
        image="my-cool-image",
    )

    expected_args = [
        "--project-directory",
        str(pathlib.Path(__file__).parent.absolute()),
        "-f",
        "custom-docker-compose.yml",
        "-f",
        "-",
    ]
    expected_stdin = f"""volumes:
    langgraph-data:
        driver: local
services:
    langgraph-redis:
        image: redis:6
        healthcheck:
            test: redis-cli ping
            interval: 5s
            timeout: 1s
            retries: 5
    langgraph-postgres:
        image: pgvector/pgvector:pg16
        ports:
            - "5433:5432"
        environment:
            POSTGRES_DB: postgres
            POSTGRES_USER: postgres
            POSTGRES_PASSWORD: postgres
        command:
            - postgres
            - -c
            - shared_preload_libraries=vector
        volumes:
            - langgraph-data:/var/lib/postgresql/data
        healthcheck:
            test: pg_isready -U postgres
            start_period: 10s
            timeout: 1s
            retries: 5
            interval: 60s
            start_interval: 1s
    langgraph-debugger:
        image: langchain/langgraph-debugger
        restart: on-failure
        depends_on:
            langgraph-postgres:
                condition: service_healthy
        ports:
            - "{debugger_port}:3968"
        environment:
            VITE_STUDIO_LOCAL_GRAPH_URL: {debugger_graph_url}
    langgraph-api:
        ports:
            - "8000:8000"
        depends_on:
            langgraph-redis:
                condition: service_healthy
            langgraph-postgres:
                condition: service_healthy
        environment:
            REDIS_URI: redis://langgraph-redis:6379
            POSTGRES_URI: {DEFAULT_POSTGRES_URI}
        image: my-cool-image
        healthcheck:
            test: python /api/healthcheck.py
            interval: 60s
            start_interval: 1s
            start_period: 10s
        
        
        develop:
            watch:
                - path: langgraph.json
                  action: rebuild
                - path: .
                  action: rebuild
                - path: ../../..
                  action: rebuild\
"""
    assert actual_args == expected_args
    assert clean_empty_lines(actual_stdin) == expected_stdin


def test_version_option() -> None:
    """Test the --version option of the CLI."""
    runner = CliRunner()
    result = runner.invoke(cli, ["--version"])

    # Verify that the command executed successfully
    assert result.exit_code == 0, "Expected exit code 0 for --version option"

    # Check that the output contains the correct version information
    assert "LangGraph CLI, version" in result.output, (
        "Expected version information in output"
    )


def test_dockerfile_command_basic() -> None:
    """Test the 'dockerfile' command with basic configuration."""
    runner = CliRunner()
    config_content = {
        "python_version": "3.11",
        "graphs": {"agent": "agent.py:graph"},
        "dependencies": ["."],
    }

    with temporary_config_folder(config_content) as temp_dir:
        save_path = temp_dir / "Dockerfile"

        result = runner.invoke(
            cli,
            ["dockerfile", str(save_path), "--config", str(temp_dir / "config.json")],
        )

        # Assert command was successful
        assert result.exit_code == 0, result.output
        assert "✅ Created: Dockerfile" in result.output

        # Check if Dockerfile was created
        assert save_path.exists()


def test_dockerfile_command_new_style_config() -> None:
    """Test `dockerfile` command with a new style config.

    This config format allows specifying agent data as a dictionary.
    {
        "graphs": {
            "agent1": {
                "path": ... # path to graph definition,
                ... # other fields
            }
        }
    }
    """
    runner = CliRunner()
    config_content = {
        "dependencies": ["./my_agent"],
        "graphs": {
            "agent": {
                "path": "./my_agent/agent.py:graph",
                "description": "This is a test agent",
            }
        },
        "env": ".env",
    }
    with temporary_config_folder(config_content) as temp_dir:
        save_path = temp_dir / "Dockerfile"
        # Add agent.py file
        agent_path = temp_dir / "my_agent" / "agent.py"
        agent_path.parent.mkdir(parents=True, exist_ok=True)
        agent_path.touch()

        result = runner.invoke(
            cli,
            ["dockerfile", str(save_path), "--config", str(temp_dir / "config.json")],
        )

        # Assert command was successful
        assert result.exit_code == 0, result.output
        assert "✅ Created: Dockerfile" in result.output

        # Check if Dockerfile was created
        assert save_path.exists()


def test_dockerfile_command_with_base_image() -> None:
    """Test the 'dockerfile' command with a base image."""
    runner = CliRunner()
    config_content = {
        "python_version": "3.11",
        "graphs": {"agent": "agent.py:graph"},
        "dependencies": ["."],
        "base_image": "langchain/langgraph-server:0.2",
    }
    with temporary_config_folder(config_content) as temp_dir:
        save_path = temp_dir / "Dockerfile"
        agent_path = temp_dir / "agent.py"
        agent_path.parent.mkdir(parents=True, exist_ok=True)
        agent_path.touch()

        result = runner.invoke(
            cli,
            ["dockerfile", str(save_path), "--config", str(temp_dir / "config.json")],
        )

        assert result.exit_code == 0, result.output
        assert "✅ Created: Dockerfile" in result.output

        assert save_path.exists()
        with open(save_path) as f:
            dockerfile = f.read()
            assert re.match("FROM langchain/langgraph-server:0.2-py3.*", dockerfile), (
                "\n".join(dockerfile.splitlines()[:3])
            )


def test_dockerfile_command_with_docker_compose() -> None:
    """Test the 'dockerfile' command with Docker Compose configuration."""
    runner = CliRunner()
    config_content = {
        "dependencies": ["./my_agent"],
        "graphs": {"agent": "./my_agent/agent.py:graph"},
        "env": ".env",
    }
    with temporary_config_folder(config_content) as temp_dir:
        save_path = temp_dir / "Dockerfile"
        # Add agent.py file
        agent_path = temp_dir / "my_agent" / "agent.py"
        agent_path.parent.mkdir(parents=True, exist_ok=True)
        agent_path.touch()

        result = runner.invoke(
            cli,
            [
                "dockerfile",
                str(save_path),
                "--config",
                str(temp_dir / "config.json"),
                "--add-docker-compose",
            ],
        )

        # Assert command was successful
        assert result.exit_code == 0
        assert "✅ Created: Dockerfile" in result.output
        assert "✅ Created: .dockerignore" in result.output
        assert "✅ Created: docker-compose.yml" in result.output
        assert (
            "✅ Created: .env" in result.output or "➖ Skipped: .env" in result.output
        )
        assert "🎉 Files generated successfully" in result.output

        # Check if Dockerfile, .dockerignore, docker-compose.yml, and .env were created
        assert save_path.exists()
        assert (temp_dir / ".dockerignore").exists()
        assert (temp_dir / "docker-compose.yml").exists()
        assert (temp_dir / ".env").exists() or "➖ Skipped: .env" in result.output


def test_dockerfile_command_with_bad_config() -> None:
    """Test the 'dockerfile' command with basic configuration."""
    runner = CliRunner()
    config_content = {
        "node_version": "20"  # Add any other necessary configuration fields
    }

    with temporary_config_folder(config_content) as temp_dir:
        save_path = temp_dir / "Dockerfile"

        result = runner.invoke(
            cli,
            ["dockerfile", str(save_path), "--config", str(temp_dir / "conf.json")],
        )

        # Assert command was successful
        assert result.exit_code == 2
        assert "conf.json' does not exist" in result.output


def test_dockerfile_command_shows_wolfi_warning() -> None:
    """Test the 'dockerfile' command shows warning when image_distro is not wolfi."""
    runner = CliRunner()
    config_content = {
        "python_version": "3.11",
        "graphs": {"agent": "agent.py:graph"},
        "dependencies": ["."],
        # No image_distro specified - should default to debian and show warning
    }

    with temporary_config_folder(config_content) as temp_dir:
        save_path = temp_dir / "Dockerfile"
        agent_path = temp_dir / "agent.py"
        agent_path.touch()

        result = runner.invoke(
            cli,
            ["dockerfile", str(save_path), "--config", str(temp_dir / "config.json")],
        )

        # Assert command was successful
        assert result.exit_code == 0, result.output

        # Check that warning is shown
        assert "Security Recommendation" in result.output
        assert "Wolfi Linux" in result.output
        assert "image_distro" in result.output
        assert "wolfi" in result.output


def test_dockerfile_command_no_wolfi_warning_when_wolfi_set() -> None:
    """Test the 'dockerfile' command does NOT show warning when image_distro is wolfi."""
    runner = CliRunner()
    config_content = {
        "python_version": "3.11",
        "graphs": {"agent": "agent.py:graph"},
        "dependencies": ["."],
        "image_distro": "wolfi",  # Explicitly set to wolfi - should not show warning
    }

    with temporary_config_folder(config_content) as temp_dir:
        save_path = temp_dir / "Dockerfile"
        agent_path = temp_dir / "agent.py"
        agent_path.touch()

        result = runner.invoke(
            cli,
            ["dockerfile", str(save_path), "--config", str(temp_dir / "config.json")],
        )

        # Assert command was successful
        assert result.exit_code == 0, result.output

        # Check that warning is NOT shown
        assert "Security Recommendation" not in result.output
        assert "Wolfi Linux" not in result.output


def test_build_command_shows_wolfi_warning() -> None:
    """Test the 'build' command shows warning when image_distro is not wolfi."""
    runner = CliRunner()
    config_content = {
        "python_version": "3.11",
        "graphs": {"agent": "agent.py:graph"},
        "dependencies": ["."],
        # No image_distro specified - should default to debian and show warning
    }

    with temporary_config_folder(config_content) as temp_dir:
        agent_path = temp_dir / "agent.py"
        agent_path.touch()

        # Mock docker command since we don't want to actually build
        with runner.isolated_filesystem():
            result = runner.invoke(
                cli,
                [
                    "build",
                    "--tag",
                    "test-image",
                    "--config",
                    str(temp_dir / "config.json"),
                ],
                catch_exceptions=True,
            )

        # The command will fail because docker isn't available or we're mocking,
        # but we should still see the warning before it fails
        assert "Security Recommendation" in result.output
        assert "Wolfi Linux" in result.output
        assert "image_distro" in result.output
        assert "wolfi" in result.output


def test_build_generate_proper_build_context():
    runner = CliRunner()
    config_content = {
        "python_version": "3.11",
        "graphs": {"agent": "agent.py:graph"},
        "dependencies": [".", "../../..", "../.."],
        "image_distro": "wolfi",
    }

    with temporary_config_folder(config_content, levels=3) as temp_dir:
        agent_path = temp_dir / "agent.py"
        agent_path.touch()

        # Mock docker command since we don't want to actually build
        with runner.isolated_filesystem():
            result = runner.invoke(
                cli,
                [
                    "build",
                    "--tag",
                    "test-image",
                    "--config",
                    str(temp_dir / "config.json"),
                ],
                catch_exceptions=True,
            )

        build_context_pattern = re.compile(r"--build-context\s+([\w-]+)=([^\s]+)")

        build_contexts = re.findall(build_context_pattern, result.output)
        assert len(build_contexts) == 2, (
            f"Expected 2 build contexts, but found {len(build_contexts)}"
        )


def test_dockerfile_command_with_api_version() -> None:
    """Test the 'dockerfile' command with --api-version flag."""
    runner = CliRunner()
    config_content = {
        "python_version": "3.11",
        "graphs": {"agent": "agent.py:graph"},
        "dependencies": ["."],
    }

    with temporary_config_folder(config_content) as temp_dir:
        save_path = temp_dir / "Dockerfile"
        agent_path = temp_dir / "agent.py"
        agent_path.touch()

        result = runner.invoke(
            cli,
            [
                "dockerfile",
                str(save_path),
                "--config",
                str(temp_dir / "config.json"),
                "--api-version",
                "0.2.74",
            ],
        )

        # Assert command was successful
        assert result.exit_code == 0, result.output
        assert "✅ Created: Dockerfile" in result.output

        # Check if Dockerfile was created and contains correct FROM line
        assert save_path.exists()
        with open(save_path) as f:
            dockerfile = f.read()
            assert "FROM langchain/langgraph-api:0.2.74-py3.11" in dockerfile


def test_dockerfile_command_with_api_version_and_base_image() -> None:
    """Test the 'dockerfile' command with both --api-version and --base-image flags."""
    runner = CliRunner()
    config_content = {
        "python_version": "3.12",
        "graphs": {"agent": "agent.py:graph"},
        "dependencies": ["."],
        "image_distro": "wolfi",
    }

    with temporary_config_folder(config_content) as temp_dir:
        save_path = temp_dir / "Dockerfile"
        agent_path = temp_dir / "agent.py"
        agent_path.touch()

        result = runner.invoke(
            cli,
            [
                "dockerfile",
                str(save_path),
                "--config",
                str(temp_dir / "config.json"),
                "--api-version",
                "1.0.0",
                "--base-image",
                "my-registry/custom-api",
            ],
        )

        # Assert command was successful
        assert result.exit_code == 0, result.output
        assert "✅ Created: Dockerfile" in result.output

        # Check if Dockerfile was created and contains correct FROM line
        assert save_path.exists()
        with open(save_path) as f:
            dockerfile = f.read()
            assert "FROM my-registry/custom-api:1.0.0-py3.12-wolfi" in dockerfile


def test_dockerfile_command_with_api_version_nodejs() -> None:
    """Test the 'dockerfile' command with --api-version flag for Node.js config."""
    runner = CliRunner()
    config_content = {
        "node_version": "20",
        "graphs": {"agent": "agent.js:graph"},
    }

    with temporary_config_folder(config_content) as temp_dir:
        save_path = temp_dir / "Dockerfile"
        agent_path = temp_dir / "agent.js"
        agent_path.touch()

        result = runner.invoke(
            cli,
            [
                "dockerfile",
                str(save_path),
                "--config",
                str(temp_dir / "config.json"),
                "--api-version",
                "0.2.74",
            ],
        )

        # Assert command was successful
        assert result.exit_code == 0, result.output
        assert "✅ Created: Dockerfile" in result.output

        # Check if Dockerfile was created and contains correct FROM line
        assert save_path.exists()
        with open(save_path) as f:
            dockerfile = f.read()
            assert "FROM langchain/langgraphjs-api:0.2.74-node20" in dockerfile


def test_build_command_with_api_version() -> None:
    """Test the 'build' command with --api-version flag."""
    runner = CliRunner()
    config_content = {
        "python_version": "3.11",
        "graphs": {"agent": "agent.py:graph"},
        "dependencies": ["."],
        "image_distro": "wolfi",  # Use wolfi to avoid warning messages
    }

    with temporary_config_folder(config_content) as temp_dir:
        agent_path = temp_dir / "agent.py"
        agent_path.touch()

        # Mock docker command since we don't want to actually build
        with runner.isolated_filesystem():
            result = runner.invoke(
                cli,
                [
                    "build",
                    "--tag",
                    "test-image",
                    "--config",
                    str(temp_dir / "config.json"),
                    "--api-version",
                    "0.2.74",
                    "--no-pull",  # Avoid pulling non-existent images
                ],
                catch_exceptions=True,
            )

        # Check that the build command is called with the correct tag
        # The output should contain the docker build command with the api_version tag
        assert "langchain/langgraph-api:0.2.74-py3.11-wolfi" in result.output


def test_build_command_with_api_version_and_base_image() -> None:
    """Test the 'build' command with both --api-version and --base-image flags."""
    runner = CliRunner()
    config_content = {
        "python_version": "3.12",
        "graphs": {"agent": "agent.py:graph"},
        "dependencies": ["."],
        "image_distro": "wolfi",  # Use wolfi to avoid warning messages
    }

    with temporary_config_folder(config_content) as temp_dir:
        agent_path = temp_dir / "agent.py"
        agent_path.touch()

        # Mock docker command since we don't want to actually build
        with runner.isolated_filesystem():
            result = runner.invoke(
                cli,
                [
                    "build",
                    "--tag",
                    "test-image",
                    "--config",
                    str(temp_dir / "config.json"),
                    "--api-version",
                    "1.0.0",
                    "--base-image",
                    "my-registry/custom-api",
                    "--no-pull",  # Avoid pulling non-existent images
                ],
                catch_exceptions=True,
            )

        # Check that the build command includes the api_version
        assert "my-registry/custom-api:1.0.0-py3.12-wolfi" in result.output


def test_prepare_args_and_stdin_with_api_version() -> None:
    """Test prepare_args_and_stdin function with api_version parameter."""
    config_path = pathlib.Path(__file__).parent / "langgraph.json"
    config = validate_config(
        Config(dependencies=["."], graphs={"agent": "agent.py:graph"})
    )
    port = 8000
    api_version = "0.2.74"

    actual_args, actual_stdin = prepare_args_and_stdin(
        capabilities=DEFAULT_DOCKER_CAPABILITIES,
        config_path=config_path,
        config=config,
        docker_compose=None,
        port=port,
        watch=False,
        api_version=api_version,
    )

    expected_args = [
        "--project-directory",
        str(pathlib.Path(__file__).parent.absolute()),
        "-f",
        "-",
    ]

    # Check that the args are correct
    assert actual_args == expected_args

    # Check that the stdin contains the correct FROM line with api_version
    assert "FROM langchain/langgraph-api:0.2.74-py3.11" in actual_stdin


def test_prepare_args_and_stdin_with_api_version_and_image() -> None:
    """Test prepare_args_and_stdin function with both api_version and image parameters."""
    config_path = pathlib.Path(__file__).parent / "langgraph.json"
    config = validate_config(
        Config(dependencies=["."], graphs={"agent": "agent.py:graph"})
    )
    port = 8000
    api_version = "0.2.74"
    image = "my-custom-image:latest"

    actual_args, actual_stdin = prepare_args_and_stdin(
        capabilities=DEFAULT_DOCKER_CAPABILITIES,
        config_path=config_path,
        config=config,
        docker_compose=None,
        port=port,
        watch=False,
        api_version=api_version,
        image=image,
    )

    # When image is provided, api_version should be ignored for the image
    # but the stdin should not contain a build section (since image is provided)
    assert "pull_policy: build" not in actual_stdin

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/cli/test_templates.py
```py
"""Unit tests for the 'new' CLI command.

This command creates a new LangGraph project using a specified template.
"""

import os
from io import BytesIO
from pathlib import Path
from tempfile import TemporaryDirectory
from unittest.mock import MagicMock, patch
from urllib import request
from zipfile import ZipFile

from click.testing import CliRunner

from langgraph_cli.cli import cli
from langgraph_cli.templates import TEMPLATE_ID_TO_CONFIG


@patch.object(request, "urlopen")
def test_create_new_with_mocked_download(mock_urlopen: MagicMock) -> None:
    """Test the 'new' CLI command with a mocked download response using urllib."""
    # Mock the response content to simulate a ZIP file
    mock_zip_content = BytesIO()
    with ZipFile(mock_zip_content, "w") as mock_zip:
        mock_zip.writestr("test-file.txt", "Test content.")

    # Create a mock response that behaves like a context manager
    mock_response = MagicMock()
    mock_response.read.return_value = mock_zip_content.getvalue()
    mock_response.__enter__.return_value = mock_response  # Setup enter context
    mock_response.status = 200

    mock_urlopen.return_value = mock_response

    with TemporaryDirectory() as temp_dir:
        runner = CliRunner()
        template = next(
            iter(TEMPLATE_ID_TO_CONFIG)
        )  # Select the first template for the test
        result = runner.invoke(cli, ["new", temp_dir, "--template", template])

        # Verify CLI command execution and success
        assert result.exit_code == 0, result.output
        assert "New project created" in result.output, (
            "Expected success message in output."
        )

        # Verify that the directory is not empty
        assert os.listdir(temp_dir), "Expected files to be created in temp directory."

        # Check for a known file in the extracted content
        extracted_files = [f.name for f in Path(temp_dir).glob("*")]
        assert "test-file.txt" in extracted_files, (
            "Expected 'test-file.txt' in the extracted content."
        )


def test_invalid_template_id() -> None:
    """Test that an invalid template ID passed via CLI results in a graceful error."""
    runner = CliRunner()
    result = runner.invoke(
        cli, ["new", "dummy_path", "--template", "invalid-template-id"]
    )

    # Verify the command failed and proper message is displayed
    assert result.exit_code != 0, "Expected non-zero exit code for invalid template."
    assert "Template 'invalid-template-id' not found" in result.output, (
        "Expected error message in output."
    )

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/unit_tests/graphs/agent.py
```py
from langgraph.func import entrypoint


@entrypoint()
def graph(state):
    return None

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/integration_tests/__init__.py
```py

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/tests/integration_tests/test_cli.py
```py
import pytest
import requests

from langgraph_cli.templates import TEMPLATE_ID_TO_CONFIG


@pytest.mark.parametrize("template_key", TEMPLATE_ID_TO_CONFIG.keys())
def test_template_urls_work(template_key: str) -> None:
    """Integration test to verify that all template URLs are reachable."""
    _, _, template_url = TEMPLATE_ID_TO_CONFIG[template_key]
    response = requests.head(template_url)
    # Returns 302 on a successful HEAD request
    assert response.status_code == 302, f"URL {template_url} is not reachable."

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/__init__.py
```py
__version__ = "0.4.7"

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/__main__.py
```py
from .cli import cli

if __name__ == "__main__":
    cli()

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/analytics.py
```py
import functools
import json
import os
import pathlib
import platform
import threading
import urllib.error
import urllib.request
from typing import Any, TypedDict

from langgraph_cli.constants import (
    DEFAULT_CONFIG,
    DEFAULT_PORT,
    SUPABASE_PUBLIC_API_KEY,
    SUPABASE_URL,
)
from langgraph_cli.version import __version__


class LogData(TypedDict):
    os: str
    os_version: str
    python_version: str
    cli_version: str
    cli_command: str
    params: dict[str, Any]


def get_anonymized_params(kwargs: dict[str, Any]) -> dict[str, bool]:
    params = {}

    # anonymize params with values
    if config := kwargs.get("config"):
        if config != pathlib.Path(DEFAULT_CONFIG).resolve():
            params["config"] = True

    if port := kwargs.get("port"):
        if port != DEFAULT_PORT:
            params["port"] = True

    if kwargs.get("docker_compose"):
        params["docker_compose"] = True

    if kwargs.get("debugger_port"):
        params["debugger_port"] = True

    if kwargs.get("postgres_uri"):
        params["postgres_uri"] = True

    # pick up exact values for boolean flags
    for boolean_param in ["recreate", "pull", "watch", "wait", "verbose"]:
        if kwargs.get(boolean_param):
            params[boolean_param] = kwargs[boolean_param]

    return params


def log_data(data: LogData) -> None:
    headers = {
        "Content-Type": "application/json",
        "apikey": SUPABASE_PUBLIC_API_KEY,
        "User-Agent": "Mozilla/5.0",
    }
    supabase_url = SUPABASE_URL

    req = urllib.request.Request(
        f"{supabase_url}/rest/v1/logs",
        data=json.dumps(data).encode("utf-8"),
        headers=headers,
        method="POST",
    )

    try:
        urllib.request.urlopen(req)
    except urllib.error.URLError:
        pass


def log_command(func):
    @functools.wraps(func)
    def decorator(*args, **kwargs):
        if os.getenv("LANGGRAPH_CLI_NO_ANALYTICS") == "1":
            return func(*args, **kwargs)

        data = {
            "os": platform.system(),
            "os_version": platform.version(),
            "python_version": platform.python_version(),
            "cli_version": __version__,
            "cli_command": func.__name__,
            "params": get_anonymized_params(kwargs),
        }

        background_thread = threading.Thread(target=log_data, args=(data,))
        background_thread.start()
        return func(*args, **kwargs)

    return decorator

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/cli.py
```py
"""CLI entrypoint for LangGraph API server."""

import os
import pathlib
import shutil
import sys
from collections.abc import Callable, Sequence

import click
import click.exceptions
from click import secho

import langgraph_cli.config
import langgraph_cli.docker
from langgraph_cli.analytics import log_command
from langgraph_cli.config import Config
from langgraph_cli.constants import DEFAULT_CONFIG, DEFAULT_PORT
from langgraph_cli.docker import DockerCapabilities
from langgraph_cli.exec import Runner, subp_exec
from langgraph_cli.progress import Progress
from langgraph_cli.templates import TEMPLATE_HELP_STRING, create_new
from langgraph_cli.util import warn_non_wolfi_distro
from langgraph_cli.version import __version__

OPT_DOCKER_COMPOSE = click.option(
    "--docker-compose",
    "-d",
    help="Advanced: Path to docker-compose.yml file with additional services to launch.",
    type=click.Path(
        exists=True,
        file_okay=True,
        dir_okay=False,
        resolve_path=True,
        path_type=pathlib.Path,
    ),
)
OPT_CONFIG = click.option(
    "--config",
    "-c",
    help="""Path to configuration file declaring dependencies, graphs and environment variables.

    \b
    Config file must be a JSON file that has the following keys:
    - "dependencies": array of dependencies for langgraph API server. Dependencies can be one of the following:
      - ".", which would look for local python packages, as well as pyproject.toml, setup.py or requirements.txt in the app directory
      - "./local_package"
      - "<package_name>
    - "graphs": mapping from graph ID to path where the compiled graph is defined, i.e. ./your_package/your_file.py:variable, where
        "variable" is an instance of langgraph.graph.graph.CompiledGraph
    - "env": (optional) path to .env file or a mapping from environment variable to its value
    - "python_version": (optional) 3.11, 3.12, or 3.13. Defaults to 3.11
    - "pip_config_file": (optional) path to pip config file
    - "dockerfile_lines": (optional) array of additional lines to add to Dockerfile following the import from parent image

    \b
    Example:
        langgraph up -c langgraph.json

    \b
    Example:
    {
        "dependencies": [
            "langchain_openai",
            "./your_package"
        ],
        "graphs": {
            "my_graph_id": "./your_package/your_file.py:variable"
        },
        "env": "./.env"
    }

    \b
    Example:
    {
        "python_version": "3.11",
        "dependencies": [
            "langchain_openai",
            "."
        ],
        "graphs": {
            "my_graph_id": "./your_package/your_file.py:variable"
        },
        "env": {
            "OPENAI_API_KEY": "secret-key"
        }
    }

    Defaults to looking for langgraph.json in the current directory.""",
    default=DEFAULT_CONFIG,
    type=click.Path(
        exists=True,
        file_okay=True,
        dir_okay=False,
        resolve_path=True,
        path_type=pathlib.Path,
    ),
)
OPT_PORT = click.option(
    "--port",
    "-p",
    type=int,
    default=DEFAULT_PORT,
    show_default=True,
    help="""
    Port to expose.

    \b
    Example:
        langgraph up --port 8000
    \b
    """,
)
OPT_RECREATE = click.option(
    "--recreate/--no-recreate",
    default=False,
    show_default=True,
    help="Recreate containers even if their configuration and image haven't changed",
)
OPT_PULL = click.option(
    "--pull/--no-pull",
    default=True,
    show_default=True,
    help="""
    Pull latest images. Use --no-pull for running the server with locally-built images.

    \b
    Example:
        langgraph up --no-pull
    \b
    """,
)
OPT_VERBOSE = click.option(
    "--verbose",
    is_flag=True,
    default=False,
    help="Show more output from the server logs",
)
OPT_WATCH = click.option("--watch", is_flag=True, help="Restart on file changes")
OPT_DEBUGGER_PORT = click.option(
    "--debugger-port",
    type=int,
    help="Pull the debugger image locally and serve the UI on specified port",
)
OPT_DEBUGGER_BASE_URL = click.option(
    "--debugger-base-url",
    type=str,
    help="URL used by the debugger to access LangGraph API. Defaults to http://127.0.0.1:[PORT]",
)

OPT_POSTGRES_URI = click.option(
    "--postgres-uri",
    help="Postgres URI to use for the database. Defaults to launching a local database",
)

OPT_API_VERSION = click.option(
    "--api-version",
    type=str,
    help="API server version to use for the base image. If unspecified, the latest version will be used.",
)


@click.group()
@click.version_option(version=__version__, prog_name="LangGraph CLI")
def cli():
    pass


@OPT_RECREATE
@OPT_PULL
@OPT_PORT
@OPT_DOCKER_COMPOSE
@OPT_CONFIG
@OPT_VERBOSE
@OPT_DEBUGGER_PORT
@OPT_DEBUGGER_BASE_URL
@OPT_WATCH
@OPT_POSTGRES_URI
@OPT_API_VERSION
@click.option(
    "--image",
    type=str,
    default=None,
    help="Docker image to use for the langgraph-api service. If specified, skips building and uses this image directly."
    " Useful if you want to test against an image already built using `langgraph build`.",
)
@click.option(
    "--base-image",
    default=None,
    help="Base image to use for the LangGraph API server. Pin to specific versions using version tags. Defaults to langchain/langgraph-api or langchain/langgraphjs-api."
    "\n\n    \b\nExamples:\n    --base-image langchain/langgraph-server:0.2.18  # Pin to a specific patch version"
    "\n    --base-image langchain/langgraph-server:0.2  # Pin to a minor version (Python)",
)
@click.option(
    "--wait",
    is_flag=True,
    help="Wait for services to start before returning. Implies --detach",
)
@cli.command(help="🚀 Launch LangGraph API server.")
@log_command
def up(
    config: pathlib.Path,
    docker_compose: pathlib.Path | None,
    port: int,
    recreate: bool,
    pull: bool,
    watch: bool,
    wait: bool,
    verbose: bool,
    debugger_port: int | None,
    debugger_base_url: str | None,
    postgres_uri: str | None,
    api_version: str | None,
    image: str | None,
    base_image: str | None,
):
    click.secho("Starting LangGraph API server...", fg="green")
    click.secho(
        """For local dev, requires env var LANGSMITH_API_KEY with access to LangSmith Deployment.
For production use, requires a license key in env var LANGGRAPH_CLOUD_LICENSE_KEY.""",
    )
    with Runner() as runner, Progress(message="Pulling...") as set:
        capabilities = langgraph_cli.docker.check_capabilities(runner)
        args, stdin = prepare(
            runner,
            capabilities=capabilities,
            config_path=config,
            docker_compose=docker_compose,
            port=port,
            pull=pull,
            watch=watch,
            verbose=verbose,
            debugger_port=debugger_port,
            debugger_base_url=debugger_base_url,
            postgres_uri=postgres_uri,
            api_version=api_version,
            image=image,
            base_image=base_image,
        )
        # add up + options
        args.extend(["up", "--remove-orphans"])
        if recreate:
            args.extend(["--force-recreate", "--renew-anon-volumes"])
            try:
                runner.run(subp_exec("docker", "volume", "rm", "langgraph-data"))
            except click.exceptions.Exit:
                pass
        if watch:
            args.append("--watch")
        if wait:
            args.append("--wait")
        else:
            args.append("--abort-on-container-exit")
        # run docker compose
        set("Building...")

        def on_stdout(line: str):
            if "unpacking to docker.io" in line:
                set("Starting...")
            elif "Application startup complete" in line:
                debugger_origin = (
                    f"http://localhost:{debugger_port}"
                    if debugger_port
                    else "https://smith.langchain.com"
                )
                debugger_base_url_query = (
                    debugger_base_url or f"http://127.0.0.1:{port}"
                )
                set("")
                sys.stdout.write(
                    f"""Ready!
- API: http://localhost:{port}
- Docs: http://localhost:{port}/docs
- LangGraph Studio: {debugger_origin}/studio/?baseUrl={debugger_base_url_query}
"""
                )
                sys.stdout.flush()
                return True

        if capabilities.compose_type == "plugin":
            compose_cmd = ["docker", "compose"]
        elif capabilities.compose_type == "standalone":
            compose_cmd = ["docker-compose"]

        runner.run(
            subp_exec(
                *compose_cmd,
                *args,
                input=stdin,
                verbose=verbose,
                on_stdout=on_stdout,
            )
        )


def _build(
    runner,
    set: Callable[[str], None],
    config: pathlib.Path,
    config_json: dict,
    base_image: str | None,
    api_version: str | None,
    pull: bool,
    tag: str,
    passthrough: Sequence[str] = (),
    install_command: str | None = None,
    build_command: str | None = None,
):
    # pull latest images
    if pull:
        runner.run(
            subp_exec(
                "docker",
                "pull",
                langgraph_cli.config.docker_tag(config_json, base_image, api_version),
                verbose=True,
            )
        )
    set("Building...")
    # apply options
    args = [
        "-f",
        "-",  # stdin
        "-t",
        tag,
    ]
    # determine build context: use current directory for JS projects, config parent for Python
    is_js_project = config_json.get("node_version") and not config_json.get(
        "python_version"
    )
    # build/install commands only apply to JS projects for now
    # without install/build command, JS projects will follow the old behavior
    if is_js_project and (build_command or install_command):
        build_context = str(pathlib.Path.cwd())
    else:
        build_context = str(config.parent)

    # apply config
    stdin, additional_contexts = langgraph_cli.config.config_to_docker(
        config,
        config_json,
        base_image,
        api_version,
        install_command,
        build_command,
        build_context,
    )
    # add additional_contexts
    if additional_contexts:
        for k, v in additional_contexts.items():
            args.extend(["--build-context", f"{k}={v}"])
    runner.run(
        subp_exec(
            "docker",
            "build",
            *args,
            *passthrough,
            build_context,
            input=stdin,
            verbose=True,
        )
    )


@OPT_CONFIG
@OPT_PULL
@click.option(
    "--tag",
    "-t",
    help="""Tag for the docker image.

    \b
    Example:
        langgraph build -t my-image

    \b
    """,
    required=True,
)
@click.option(
    "--base-image",
    help="Base image to use for the LangGraph API server. Pin to specific versions using version tags. Defaults to langchain/langgraph-api or langchain/langgraphjs-api."
    "\n\n    \b\nExamples:\n    --base-image langchain/langgraph-server:0.2.18  # Pin to a specific patch version"
    "\n    --base-image langchain/langgraph-server:0.2  # Pin to a minor version (Python)",
)
@OPT_API_VERSION
@click.option(
    "--install-command",
    help="Custom install command to run from the build context root. If not provided, auto-detects based on package manager files.",
)
@click.option(
    "--build-command",
    help="Custom build command to run from the langgraph.json directory. If not provided, uses default build process.",
)
@click.argument("docker_build_args", nargs=-1, type=click.UNPROCESSED)
@cli.command(
    help="📦 Build LangGraph API server Docker image.",
    context_settings=dict(
        ignore_unknown_options=True,
    ),
)
@log_command
def build(
    config: pathlib.Path,
    docker_build_args: Sequence[str],
    base_image: str | None,
    api_version: str | None,
    pull: bool,
    tag: str,
    install_command: str | None,
    build_command: str | None,
):
    with Runner() as runner, Progress(message="Pulling...") as set:
        if shutil.which("docker") is None:
            raise click.UsageError("Docker not installed") from None
        config_json = langgraph_cli.config.validate_config_file(config)
        warn_non_wolfi_distro(config_json)
        _build(
            runner,
            set,
            config,
            config_json,
            base_image,
            api_version,
            pull,
            tag,
            docker_build_args,
            install_command,
            build_command,
        )


def _get_docker_ignore_content() -> str:
    """Return the content of a .dockerignore file.

    This file is used to exclude files and directories from the Docker build context.

    It may be overly broad, but it's better to be safe than sorry.

    The main goal is to exclude .env files by default.
    """
    return """\
# Ignore node_modules and other dependency directories
node_modules
bower_components
vendor

# Ignore logs and temporary files
*.log
*.tmp
*.swp

# Ignore .env files and other environment files
.env
.env.*
*.local

# Ignore git-related files
.git
.gitignore

# Ignore Docker-related files and configs
.dockerignore
docker-compose.yml

# Ignore build and cache directories
dist
build
.cache
__pycache__

# Ignore IDE and editor configurations
.vscode
.idea
*.sublime-project
*.sublime-workspace
.DS_Store  # macOS-specific

# Ignore test and coverage files
coverage
*.coverage
*.test.js
*.spec.js
tests
"""


@OPT_CONFIG
@click.argument("save_path", type=click.Path(resolve_path=True))
@cli.command(
    help="🐳 Generate a Dockerfile for the LangGraph API server, with Docker Compose options."
)
@click.option(
    # Add a flag for adding a docker-compose.yml file as part of the output
    "--add-docker-compose",
    help=(
        "Add additional files for running the LangGraph API server with "
        "docker-compose. These files include a docker-compose.yml, .env file, "
        "and a .dockerignore file."
    ),
    is_flag=True,
)
@click.option(
    "--base-image",
    help="Base image to use for the LangGraph API server. Pin to specific versions using version tags. Defaults to langchain/langgraph-api or langchain/langgraphjs-api."
    "\n\n    \b\nExamples:\n    --base-image langchain/langgraph-server:0.2.18  # Pin to a specific patch version"
    "\n    --base-image langchain/langgraph-server:0.2  # Pin to a minor version (Python)",
)
@OPT_API_VERSION
@log_command
def dockerfile(
    save_path: str,
    config: pathlib.Path,
    add_docker_compose: bool,
    base_image: str | None = None,
    api_version: str | None = None,
) -> None:
    save_path = pathlib.Path(save_path).absolute()
    secho(f"🔍 Validating configuration at path: {config}", fg="yellow")
    config_json = langgraph_cli.config.validate_config_file(config)
    warn_non_wolfi_distro(config_json)
    secho("✅ Configuration validated!", fg="green")

    secho(f"📝 Generating Dockerfile at {save_path}", fg="yellow")
    dockerfile, additional_contexts = langgraph_cli.config.config_to_docker(
        config,
        config_json,
        base_image=base_image,
        api_version=api_version,
    )
    with open(str(save_path), "w", encoding="utf-8") as f:
        f.write(dockerfile)
    secho("✅ Created: Dockerfile", fg="green")

    if additional_contexts:
        additional_contexts_str = ",".join(
            f"{k}={v}" for k, v in additional_contexts.items()
        )
        secho(
            f"""📝 Run docker build with these additional build contexts `--build-context {additional_contexts_str}`""",
            fg="yellow",
        )

    if add_docker_compose:
        # Add docker compose and related files
        # Add .dockerignore file in the same directory as the Dockerfile
        with open(str(save_path.parent / ".dockerignore"), "w", encoding="utf-8") as f:
            f.write(_get_docker_ignore_content())
        secho("✅ Created: .dockerignore", fg="green")

        # Generate a docker-compose.yml file
        path = str(save_path.parent / "docker-compose.yml")
        with open(path, "w", encoding="utf-8") as f:
            with Runner() as runner:
                capabilities = langgraph_cli.docker.check_capabilities(runner)

            compose_dict = langgraph_cli.docker.compose_as_dict(
                capabilities,
                port=8123,
                base_image=base_image,
            )
            # Add .env file to the docker-compose.yml for the langgraph-api service
            compose_dict["services"]["langgraph-api"]["env_file"] = [".env"]
            # Add the Dockerfile to the build context
            compose_dict["services"]["langgraph-api"]["build"] = {
                "context": ".",
                "dockerfile": save_path.name,
            }
            # Add the base_image as build arg if provided
            if base_image:
                compose_dict["services"]["langgraph-api"]["build"]["args"] = {
                    "BASE_IMAGE": base_image
                }
            f.write(langgraph_cli.docker.dict_to_yaml(compose_dict))
            secho("✅ Created: docker-compose.yml", fg="green")

        # Check if the .env file exists in the same directory as the Dockerfile
        if not (save_path.parent / ".env").exists():
            # Also add an empty .env file
            with open(str(save_path.parent / ".env"), "w", encoding="utf-8") as f:
                f.writelines(
                    [
                        "# Uncomment the following line to add your LangSmith API key",
                        "\n",
                        "# LANGSMITH_API_KEY=your-api-key",
                        "\n",
                        "# Or if you have a LangSmith Deployment license key, "
                        "then uncomment the following line: ",
                        "\n",
                        "# LANGGRAPH_CLOUD_LICENSE_KEY=your-license-key",
                        "\n",
                        "# Add any other environment variables go below...",
                    ]
                )

            secho("✅ Created: .env", fg="green")
        else:
            # Do nothing since the .env file already exists. Not a great
            # idea to overwrite in case the user has added custom env vars set
            # in the .env file already.
            secho("➖ Skipped: .env. It already exists!", fg="yellow")

    secho(
        f"🎉 Files generated successfully at path {save_path.parent}!",
        fg="cyan",
        bold=True,
    )


@click.option(
    "--host",
    default="127.0.0.1",
    help="Network interface to bind the development server to. Default 127.0.0.1 is recommended for security. Only use 0.0.0.0 in trusted networks",
)
@click.option(
    "--port",
    default=2024,
    type=int,
    help="Port number to bind the development server to. Example: langgraph dev --port 8000",
)
@click.option(
    "--no-reload",
    is_flag=True,
    help="Disable automatic reloading when code changes are detected",
)
@click.option(
    "--config",
    type=click.Path(exists=True),
    default="langgraph.json",
    help="Path to configuration file declaring dependencies, graphs and environment variables",
)
@click.option(
    "--n-jobs-per-worker",
    default=None,
    type=int,
    help="Maximum number of concurrent jobs each worker process can handle. Default: 10",
)
@click.option(
    "--no-browser",
    is_flag=True,
    help="Skip automatically opening the browser when the server starts",
)
@click.option(
    "--debug-port",
    default=None,
    type=int,
    help="Enable remote debugging by listening on specified port. Requires debugpy to be installed",
)
@click.option(
    "--wait-for-client",
    is_flag=True,
    help="Wait for a debugger client to connect to the debug port before starting the server",
    default=False,
)
@click.option(
    "--studio-url",
    type=str,
    default=None,
    help="URL of the LangGraph Studio instance to connect to. Defaults to https://smith.langchain.com",
)
@click.option(
    "--allow-blocking",
    is_flag=True,
    help="Don't raise errors for synchronous I/O blocking operations in your code.",
    default=False,
)
@click.option(
    "--tunnel",
    is_flag=True,
    help="Expose the local server via a public tunnel (in this case, Cloudflare) "
    "for remote frontend access. This avoids issues with browsers "
    "or networks blocking localhost connections.",
    default=False,
)
@click.option(
    "--server-log-level",
    type=str,
    default="WARNING",
    help="Set the log level for the API server.",
)
@cli.command(
    "dev",
    help="🏃‍♀️‍➡️ Run LangGraph API server in development mode with hot reloading and debugging support",
)
@log_command
def dev(
    host: str,
    port: int,
    no_reload: bool,
    config: str,
    n_jobs_per_worker: int | None,
    no_browser: bool,
    debug_port: int | None,
    wait_for_client: bool,
    studio_url: str | None,
    allow_blocking: bool,
    tunnel: bool,
    server_log_level: str,
):
    """CLI entrypoint for running the LangGraph API server."""
    try:
        from langgraph_api.cli import run_server  # type: ignore
    except ImportError:
        py_version_msg = ""
        if sys.version_info < (3, 11):
            py_version_msg = (
                "\n\nNote: The in-mem server requires Python 3.11 or higher to be installed."
                f" You are currently using Python {sys.version_info.major}.{sys.version_info.minor}."
                ' Please upgrade your Python version before installing "langgraph-cli[inmem]".'
            )
        try:
            from importlib import util

            if not util.find_spec("langgraph_api"):
                raise click.UsageError(
                    "Required package 'langgraph-api' is not installed.\n"
                    "Please install it with:\n\n"
                    '    pip install -U "langgraph-cli[inmem]"'
                    f"{py_version_msg}"
                ) from None
        except ImportError:
            raise click.UsageError(
                "Could not verify package installation. Please ensure Python is up to date and\n"
                "langgraph-cli is installed with the 'inmem' extra: pip install -U \"langgraph-cli[inmem]\""
                f"{py_version_msg}"
            ) from None
        raise click.UsageError(
            "Could not import run_server. This likely means your installation is incomplete.\n"
            "Please ensure langgraph-cli is installed with the 'inmem' extra: pip install -U \"langgraph-cli[inmem]\""
            f"{py_version_msg}"
        ) from None

    config_json = langgraph_cli.config.validate_config_file(pathlib.Path(config))
    if config_json.get("node_version"):
        raise click.UsageError(
            "In-mem server for JS graphs is not supported in this version of the LangGraph CLI. Please use `npx @langchain/langgraph-cli` instead."
        ) from None

    cwd = os.getcwd()
    sys.path.append(cwd)
    dependencies = config_json.get("dependencies", [])
    for dep in dependencies:
        dep_path = pathlib.Path(cwd) / dep
        if dep_path.is_dir() and dep_path.exists():
            sys.path.append(str(dep_path))

    graphs = config_json.get("graphs", {})

    run_server(
        host,
        port,
        not no_reload,
        graphs,
        n_jobs_per_worker=n_jobs_per_worker,
        open_browser=not no_browser,
        debug_port=debug_port,
        env=config_json.get("env"),
        store=config_json.get("store"),
        wait_for_client=wait_for_client,
        auth=config_json.get("auth"),
        http=config_json.get("http"),
        ui=config_json.get("ui"),
        ui_config=config_json.get("ui_config"),
        studio_url=studio_url,
        allow_blocking=allow_blocking,
        tunnel=tunnel,
        server_level=server_log_level,
    )


@click.argument("path", required=False)
@click.option(
    "--template",
    type=str,
    help=TEMPLATE_HELP_STRING,
)
@cli.command("new", help="🌱 Create a new LangGraph project from a template.")
@log_command
def new(path: str | None, template: str | None) -> None:
    """Create a new LangGraph project from a template."""
    return create_new(path, template)


def prepare_args_and_stdin(
    *,
    capabilities: DockerCapabilities,
    config_path: pathlib.Path,
    config: Config,
    docker_compose: pathlib.Path | None,
    port: int,
    watch: bool,
    debugger_port: int | None = None,
    debugger_base_url: str | None = None,
    postgres_uri: str | None = None,
    api_version: str | None = None,
    # Like "my-tag" (if you already built it locally)
    image: str | None = None,
    # Like "langchain/langgraphjs-api" or "langchain/langgraph-api
    base_image: str | None = None,
) -> tuple[list[str], str]:
    assert config_path.exists(), f"Config file not found: {config_path}"
    # prepare args
    stdin = langgraph_cli.docker.compose(
        capabilities,
        port=port,
        debugger_port=debugger_port,
        debugger_base_url=debugger_base_url,
        postgres_uri=postgres_uri,
        image=image,  # Pass image to compose YAML generator
        base_image=base_image,
        api_version=api_version,
    )
    args = [
        "--project-directory",
        str(config_path.parent),
    ]
    # apply options
    if docker_compose:
        args.extend(["-f", str(docker_compose)])
    args.extend(["-f", "-"])  # stdin
    # apply config
    stdin += langgraph_cli.config.config_to_compose(
        config_path,
        config,
        watch=watch,
        base_image=langgraph_cli.config.default_base_image(config),
        api_version=api_version,
        image=image,
    )
    return args, stdin


def prepare(
    runner,
    *,
    capabilities: DockerCapabilities,
    config_path: pathlib.Path,
    docker_compose: pathlib.Path | None,
    port: int,
    pull: bool,
    watch: bool,
    verbose: bool,
    debugger_port: int | None = None,
    debugger_base_url: str | None = None,
    postgres_uri: str | None = None,
    api_version: str | None = None,
    image: str | None = None,
    base_image: str | None = None,
) -> tuple[list[str], str]:
    """Prepare the arguments and stdin for running the LangGraph API server."""
    config_json = langgraph_cli.config.validate_config_file(config_path)
    warn_non_wolfi_distro(config_json)
    # pull latest images
    if pull:
        runner.run(
            subp_exec(
                "docker",
                "pull",
                langgraph_cli.config.docker_tag(config_json, base_image, api_version),
                verbose=verbose,
            )
        )

    args, stdin = prepare_args_and_stdin(
        capabilities=capabilities,
        config_path=config_path,
        config=config_json,
        docker_compose=docker_compose,
        port=port,
        watch=watch,
        debugger_port=debugger_port,
        debugger_base_url=debugger_base_url or f"http://127.0.0.1:{port}",
        postgres_uri=postgres_uri,
        api_version=api_version,
        image=image,
        base_image=base_image,
    )
    return args, stdin

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/config.py
```py
import json
import os
import pathlib
import re
import textwrap
from collections import Counter
from typing import Literal, NamedTuple

import click

from langgraph_cli.schemas import Config, Distros

MIN_NODE_VERSION = "20"
DEFAULT_NODE_VERSION = "20"

MIN_PYTHON_VERSION = "3.11"
DEFAULT_PYTHON_VERSION = "3.11"

DEFAULT_IMAGE_DISTRO = "debian"


_BUILD_TOOLS = ("pip", "setuptools", "wheel")


def _get_pip_cleanup_lines(
    install_cmd: str,
    to_uninstall: tuple[str] | None,
    pip_installer: Literal["uv", "pip"],
) -> str:
    commands = [
        f"""# -- Ensure user deps didn't inadvertently overwrite langgraph-api
RUN mkdir -p /api/langgraph_api /api/langgraph_runtime /api/langgraph_license && \
touch /api/langgraph_api/__init__.py /api/langgraph_runtime/__init__.py /api/langgraph_license/__init__.py
RUN PYTHONDONTWRITEBYTECODE=1 {install_cmd} --no-cache-dir --no-deps -e /api
# -- End of ensuring user deps didn't inadvertently overwrite langgraph-api --
# -- Removing build deps from the final image ~<:===~~~ --"""
    ]
    if to_uninstall:
        for pack in to_uninstall:
            if pack not in _BUILD_TOOLS:
                raise ValueError(
                    f"Invalid build tool: {pack}; must be one of {', '.join(_BUILD_TOOLS)}"
                )
        packs_str = " ".join(sorted(to_uninstall))
        commands.append(f"RUN pip uninstall -y {packs_str}")
        # Ensure the directories are removed entirely
        packages_rm = " ".join(
            f"/usr/local/lib/python*/site-packages/{pack}*" for pack in to_uninstall
        )
        if "pip" in to_uninstall:
            packages_rm += ' && find /usr/local/bin -name "pip*" -delete || true'
        commands.append(f"RUN rm -rf {packages_rm}")
        wolfi_packages_rm = " ".join(
            f"/usr/lib/python*/site-packages/{pack}*" for pack in to_uninstall
        )
        if "pip" in to_uninstall:
            wolfi_packages_rm += ' && find /usr/bin -name "pip*" -delete || true'
        commands.append(f"RUN rm -rf {wolfi_packages_rm}")
        if pip_installer == "uv":
            commands.append(
                f"RUN uv pip uninstall --system {packs_str} && rm /usr/bin/uv /usr/bin/uvx"
            )
    else:
        if pip_installer == "uv":
            commands.append(
                "RUN rm /usr/bin/uv /usr/bin/uvx\n# -- End of build deps removal --"
            )
    return "\n".join(commands)


def _parse_version(version_str: str) -> tuple[int, int]:
    """Parse a version string into a tuple of (major, minor)."""
    try:
        major, minor = map(int, version_str.split("-")[0].split("."))
        return (major, minor)
    except ValueError:
        raise click.UsageError(f"Invalid version format: {version_str}") from None


def _parse_node_version(version_str: str) -> int:
    """Parse a Node.js version string into a major version number."""
    try:
        if "." in version_str:
            raise ValueError("Node.js version must be major version only")
        return int(version_str)
    except ValueError:
        raise click.UsageError(
            f"Invalid Node.js version format: {version_str}. "
            "Use major version only (e.g., '20')."
        ) from None


def _is_node_graph(spec: str | dict) -> bool:
    """Check if a graph is a Node.js graph based on the file extension."""
    if isinstance(spec, dict):
        spec = spec.get("path")

    file_path = spec.split(":")[0]
    file_ext = os.path.splitext(file_path)[1]

    return file_ext in [
        ".ts",
        ".mts",
        ".cts",
        ".js",
        ".mjs",
        ".cjs",
    ]


def validate_config(config: Config) -> Config:
    """Validate a configuration dictionary."""

    graphs = config.get("graphs", {})

    some_node = any(_is_node_graph(spec) for spec in graphs.values())
    some_python = any(not _is_node_graph(spec) for spec in graphs.values())

    node_version = config.get(
        "node_version", DEFAULT_NODE_VERSION if some_node else None
    )
    python_version = config.get(
        "python_version", DEFAULT_PYTHON_VERSION if some_python else None
    )

    image_distro = config.get("image_distro", DEFAULT_IMAGE_DISTRO)
    internal_docker_tag = config.get("_INTERNAL_docker_tag")
    api_version = config.get("api_version")
    if internal_docker_tag:
        if api_version:
            raise click.UsageError(
                "Cannot specify both _INTERNAL_docker_tag and api_version."
            )
    if api_version:
        try:
            parts = tuple(map(int, api_version.split("-")[0].split(".")))
            if len(parts) > 3:
                raise ValueError(
                    "Version must be major or major.minor or major.minor.patch."
                )
        except TypeError:
            raise click.UsageError(f"Invalid version format: {api_version}") from None

    config = {
        "node_version": node_version,
        "python_version": python_version,
        "pip_config_file": config.get("pip_config_file"),
        "pip_installer": config.get("pip_installer", "auto"),
        "base_image": config.get("base_image"),
        "image_distro": image_distro,
        "dependencies": config.get("dependencies", []),
        "dockerfile_lines": config.get("dockerfile_lines", []),
        "graphs": config.get("graphs", {}),
        "env": config.get("env", {}),
        "store": config.get("store"),
        "auth": config.get("auth"),
        "http": config.get("http"),
        "checkpointer": config.get("checkpointer"),
        "ui": config.get("ui"),
        "ui_config": config.get("ui_config"),
        "keep_pkg_tools": config.get("keep_pkg_tools"),
    }
    if internal_docker_tag:
        config["_INTERNAL_docker_tag"] = internal_docker_tag
    if api_version:
        config["api_version"] = api_version

    if config.get("node_version"):
        node_version = config["node_version"]
        try:
            major = _parse_node_version(node_version)
            min_major = _parse_node_version(MIN_NODE_VERSION)
            if major < min_major:
                raise click.UsageError(
                    f"Node.js version {node_version} is not supported. "
                    f"Minimum required version is {MIN_NODE_VERSION}."
                )
        except ValueError as e:
            raise click.UsageError(str(e)) from None

    if config.get("python_version"):
        pyversion = config["python_version"]
        if not pyversion.count(".") == 1 or not all(
            part.isdigit() for part in pyversion.split("-")[0].split(".")
        ):
            raise click.UsageError(
                f"Invalid Python version format: {pyversion}. "
                "Use 'major.minor' format (e.g., '3.11'). "
                "Patch version cannot be specified."
            )
        if _parse_version(pyversion) < _parse_version(MIN_PYTHON_VERSION):
            raise click.UsageError(
                f"Python version {pyversion} is not supported. "
                f"Minimum required version is {MIN_PYTHON_VERSION}."
            )

        if not config["dependencies"]:
            raise click.UsageError(
                "No dependencies found in config. "
                "Add at least one dependency to 'dependencies' list."
            )

    if not config.get("graphs"):
        raise click.UsageError(
            "No graphs found in config. Add at least one graph to 'graphs' dictionary."
        )

    # Validate image_distro config
    if image_distro := config.get("image_distro"):
        if image_distro not in Distros.__args__:
            raise click.UsageError(
                f"Invalid image_distro: '{image_distro}'. "
                "Must be one of 'debian', 'bullseye', or 'bookworm'."
            )

    if pip_installer := config.get("pip_installer"):
        if pip_installer not in ["auto", "pip", "uv"]:
            raise click.UsageError(
                f"Invalid pip_installer: '{pip_installer}'. "
                "Must be 'auto', 'pip', or 'uv'."
            )

    # Validate auth config
    if auth_conf := config.get("auth"):
        if "path" in auth_conf:
            if ":" not in auth_conf["path"]:
                raise ValueError(
                    f"Invalid auth.path format: '{auth_conf['path']}'. "
                    "Must be in format './path/to/file.py:attribute_name'"
                )
    if http_conf := config.get("http"):
        if "app" in http_conf:
            if ":" not in http_conf["app"]:
                raise ValueError(
                    f"Invalid http.app format: '{http_conf['app']}'. "
                    "Must be in format './path/to/file.py:attribute_name'"
                )
    if keep_pkg_tools := config.get("keep_pkg_tools"):
        if isinstance(keep_pkg_tools, list):
            for tool in keep_pkg_tools:
                if tool not in _BUILD_TOOLS:
                    raise ValueError(
                        f"Invalid keep_pkg_tools: '{tool}'. "
                        "Must be one of 'pip', 'setuptools', 'wheel'."
                    )
        elif keep_pkg_tools is True:
            pass
        else:
            raise ValueError(
                f"Invalid keep_pkg_tools: '{keep_pkg_tools}'. "
                "Must be bool or list[str] (with values"
                " 'pip', 'setuptools', and/or 'wheel')."
            )
    return config


def validate_config_file(config_path: pathlib.Path) -> Config:
    """Load and validate a configuration file."""
    with open(config_path) as f:
        config = json.load(f)
    validated = validate_config(config)
    # Enforce the package.json doesn't enforce an
    # incompatible Node.js version
    if validated.get("node_version"):
        package_json_path = config_path.parent / "package.json"
        if package_json_path.is_file():
            try:
                with open(package_json_path) as f:
                    package_json = json.load(f)
                    if "engines" in package_json:
                        engines = package_json["engines"]
                        if any(engine != "node" for engine in engines.keys()):
                            raise click.UsageError(
                                "Only 'node' engine is supported in package.json engines."
                                f" Got engines: {list(engines.keys())}"
                            )
                        if engines:
                            node_version = engines["node"]
                            try:
                                major = _parse_node_version(node_version)
                                min_major = _parse_node_version(MIN_NODE_VERSION)
                                if major < min_major:
                                    raise click.UsageError(
                                        f"Node.js version in package.json engines must be >= {MIN_NODE_VERSION} "
                                        f"(major version only), got '{node_version}'. Minor/patch versions "
                                        "(like '20.x.y') are not supported to prevent deployment issues "
                                        "when new Node.js versions are released."
                                    )
                            except ValueError as e:
                                raise click.UsageError(str(e)) from None

            except json.JSONDecodeError:
                raise click.UsageError(
                    "Invalid package.json found in langgraph "
                    f"config directory {package_json_path}: file is not valid JSON"
                ) from None
    return validated


class LocalDeps(NamedTuple):
    """A container for referencing and managing local Python dependencies.

    A "local dependency" is any entry in the config's `dependencies` list
    that starts with "." (dot), denoting a relative path
    to a local directory containing Python code.

    For each local dependency, the system inspects its directory to
    determine how it should be installed inside the Docker container.

    Specifically, we detect:

    - **Real packages**: Directories containing a `pyproject.toml` or a `setup.py`.
      These can be installed with pip as a regular Python package.
    - **Faux packages**: Directories that do not include a `pyproject.toml` or
      `setup.py` but do contain Python files and possibly an `__init__.py`. For
      these, the code dynamically generates a minimal `pyproject.toml` in the
      Docker image so that they can still be installed with pip.
    - **Requirements files**: If a local dependency directory
      has a `requirements.txt`, it is tracked so that those dependencies
      can be installed within the Docker container before installing the local package.

    Attributes:
        pip_reqs: A list of (host_requirements_path, container_requirements_path)
            tuples. Each entry points to a local `requirements.txt` file and where
            it should be placed inside the Docker container before running `pip install`.

        real_pkgs: A dictionary mapping a local directory path (host side) to a
            tuple of (dependency_string, container_package_path). These directories
            contain the necessary files (e.g., `pyproject.toml` or `setup.py`) to be
            installed as a standard Python package with pip.

        faux_pkgs: A dictionary mapping a local directory path (host side) to a
            tuple of (dependency_string, container_package_path). For these
            directories—called "faux packages"—the code will generate a minimal
            `pyproject.toml` inside the Docker image. This ensures that pip
            recognizes them as installable packages, even though they do not
            natively include packaging metadata.

        working_dir: The path inside the Docker container to use as the working
            directory. If the local dependency `"."` is present in the config, this
            field captures the path where that dependency will appear in the
            container (e.g., `/deps/<name>` or similar). Otherwise, it may be `None`.

        additional_contexts: A list of paths to directories that contain local
            dependencies in parent directories. These directories are added to the
            Docker build context to ensure that the Dockerfile can access them.
    """

    pip_reqs: list[tuple[pathlib.Path, str]]
    real_pkgs: dict[pathlib.Path, tuple[str, str]]
    faux_pkgs: dict[pathlib.Path, tuple[str, str]]
    # if . is in dependencies, use it as working_dir
    working_dir: str | None = None
    # if there are local dependencies in parent directories, use additional_contexts
    additional_contexts: list[pathlib.Path] = None


def _assemble_local_deps(config_path: pathlib.Path, config: Config) -> LocalDeps:
    config_path = config_path.resolve()
    # ensure reserved package names are not used
    reserved = {
        "src",
        "langgraph-api",
        "langgraph_api",
        "langgraph",
        "langchain-core",
        "langchain_core",
        "pydantic",
        "orjson",
        "fastapi",
        "uvicorn",
        "psycopg",
        "httpx",
        "langsmith",
    }
    counter = Counter()

    def check_reserved(name: str, ref: str):
        if name in reserved:
            raise ValueError(
                f"Package name '{name}' used in local dep '{ref}' is reserved. "
                "Rename the directory."
            )
        reserved.add(name)

    pip_reqs = []
    real_pkgs = {}
    faux_pkgs = {}
    working_dir: str | None = None
    additional_contexts: list[pathlib.Path] = []

    for local_dep in config["dependencies"]:
        if not local_dep.startswith("."):
            # If the dependency is not a local path, skip it
            continue

        # Verify that the local dependency can be resolved
        # (e.g., this would raise an informative error if a user mistyped a path).
        resolved = (config_path.parent / local_dep).resolve()

        # validate local dependency
        if not resolved.exists():
            raise FileNotFoundError(f"Could not find local dependency: {resolved}")
        elif not resolved.is_dir():
            raise NotADirectoryError(
                f"Local dependency must be a directory: {resolved}"
            )
        elif resolved == config_path.parent:
            pass
        elif config_path.parent not in resolved.parents:
            additional_contexts.append(resolved)

        # Check for pyproject.toml or setup.py
        # If found, treat as a real package, if not treat as a faux package.
        # For faux packages, we'll also check for presence of requirements.txt.
        files = os.listdir(resolved)
        if "pyproject.toml" in files or "setup.py" in files:
            # real package

            # assign a unique folder name
            container_name = resolved.name
            if counter[container_name] > 0:
                container_name += f"_{counter[container_name]}"
            counter[container_name] += 1
            # add to deps
            real_pkgs[resolved] = (local_dep, container_name)
            # set working_dir
            if local_dep == ".":
                working_dir = f"/deps/{container_name}"
        else:
            # We could not find a pyproject.toml or setup.py, so treat as a faux package
            if any(file == "__init__.py" for file in files):
                # flat layout
                if "-" in resolved.name:
                    raise ValueError(
                        f"Package name '{resolved.name}' contains a hyphen. "
                        "Rename the directory to use it as flat-layout package."
                    )
                check_reserved(resolved.name, local_dep)
                container_path = f"/deps/outer-{resolved.name}/{resolved.name}"
            else:
                # src layout
                container_path = f"/deps/outer-{resolved.name}/src"
                for file in files:
                    rfile = resolved / file
                    if (
                        rfile.is_dir()
                        and file != "__pycache__"
                        and not file.startswith(".")
                    ):
                        try:
                            for subfile in os.listdir(rfile):
                                if subfile.endswith(".py"):
                                    check_reserved(file, local_dep)
                                    break
                        except PermissionError:
                            pass
            faux_pkgs[resolved] = (local_dep, container_path)
            if local_dep == ".":
                working_dir = container_path

            # If the faux package has a requirements.txt, we'll add
            # the path to the list of requirements to install.
            if "requirements.txt" in files:
                rfile = resolved / "requirements.txt"
                pip_reqs.append(
                    (
                        rfile,
                        f"{container_path}/requirements.txt",
                    )
                )

    return LocalDeps(pip_reqs, real_pkgs, faux_pkgs, working_dir, additional_contexts)


def _update_graph_paths(
    config_path: pathlib.Path, config: Config, local_deps: LocalDeps
) -> None:
    """Remap each graph's import path to the correct in-container path.

    The config may contain entries in `graphs` that look like this:
        {
            "my_graph": "./mygraphs/main.py:graph_function"
        }
    or
        {
            "my_graph": "./src/some_subdir/my_file.py:my_graph"
        }
    which indicate a local file (on the host) followed by a colon and a
    callable/object attribute within that file.

    During the Docker build, local directories are copied into special
    `/deps/` subdirectories, so they can be installed or referenced in
    the container. This function updates each graph's import path to
    reflect its new location **inside** the Docker container.

    Paths inside the container must be POSIX-style paths (even if
    the host system is Windows).

    Args:
        config_path: The path to the config file (e.g. `langgraph.json`).
        config: The validated configuration dictionary.
        local_deps: An object containing references to local dependencies:
            - real Python packages (with a `pyproject.toml` or `setup.py`)
            - “faux” packages that need minimal metadata to be installable
            - potential `requirements.txt` for local dependencies
            - container work directory (if "." is in `dependencies`)

    Raises:
        ValueError: If the import string is not in the format `<module>:<attribute>`
                    or if the referenced local file is not found in `dependencies`.
        FileNotFoundError: If the local file (module) does not actually exist on disk.
        IsADirectoryError: If `module_str` points to a directory instead of a file.
    """
    for graph_id, data in config["graphs"].items():
        if isinstance(data, dict):
            # Then we're looking for a 'path' key
            if "path" not in data:
                raise ValueError(
                    f"Graph '{graph_id}' must contain a 'path' key if "
                    f" it is a dictionary."
                )
            import_str = data["path"]
        elif isinstance(data, str):
            import_str = data
        else:
            raise ValueError(
                f"Graph '{graph_id}' must be a string or a dictionary with a 'path' key."
            )

        module_str, _, attr_str = import_str.partition(":")
        if not module_str or not attr_str:
            message = (
                'Import string "{import_str}" must be in format "<module>:<attribute>".'
            )
            raise ValueError(message.format(import_str=import_str))

        # Check for either forward slash or backslash in the module string
        # to determine if it's a file path.
        if "/" in module_str or "\\" in module_str:
            # Resolve the local path properly on the current OS
            resolved = (config_path.parent / module_str).resolve()
            if not resolved.exists():
                raise FileNotFoundError(f"Could not find local module: {resolved}")
            elif not resolved.is_file():
                raise IsADirectoryError(f"Local module must be a file: {resolved}")
            else:
                for path in local_deps.real_pkgs:
                    if resolved.is_relative_to(path):
                        container_path = (
                            pathlib.Path("/deps")
                            / path.name
                            / resolved.relative_to(path)
                        )
                        module_str = container_path.as_posix()
                        break
                else:
                    for faux_pkg, (_, destpath) in local_deps.faux_pkgs.items():
                        if resolved.is_relative_to(faux_pkg):
                            container_subpath = resolved.relative_to(faux_pkg)
                            # Construct the final path, ensuring POSIX style
                            module_str = f"{destpath}/{container_subpath.as_posix()}"
                            break
                    else:
                        raise ValueError(
                            f"Module '{import_str}' not found in 'dependencies' list. "
                            "Add its containing package to 'dependencies' list."
                        )
            # update the config
            if isinstance(data, dict):
                config["graphs"][graph_id]["path"] = f"{module_str}:{attr_str}"
            else:
                config["graphs"][graph_id] = f"{module_str}:{attr_str}"


def _update_auth_path(
    config_path: pathlib.Path, config: Config, local_deps: LocalDeps
) -> None:
    """Update auth.path to use Docker container paths."""
    auth_conf = config.get("auth")
    if not auth_conf or not (path_str := auth_conf.get("path")):
        return

    module_str, sep, attr_str = path_str.partition(":")
    if not sep or not module_str.startswith("."):
        return  # Already validated or absolute path

    resolved = config_path.parent / module_str
    if not resolved.exists():
        raise FileNotFoundError(f"Auth file not found: {resolved} (from {path_str})")
    if not resolved.is_file():
        raise IsADirectoryError(f"Auth path must be a file: {resolved}")

    # Check faux packages first (higher priority)
    for faux_path, (_, destpath) in local_deps.faux_pkgs.items():
        if resolved.is_relative_to(faux_path):
            new_path = f"{destpath}/{resolved.relative_to(faux_path)}:{attr_str}"
            auth_conf["path"] = new_path
            return

    # Check real packages
    for real_path in local_deps.real_pkgs:
        if resolved.is_relative_to(real_path):
            new_path = (
                f"/deps/{real_path.name}/{resolved.relative_to(real_path)}:{attr_str}"
            )
            auth_conf["path"] = new_path
            return

    raise ValueError(
        f"Auth file '{resolved}' not covered by dependencies.\n"
        "Add its parent directory to the 'dependencies' array in your config.\n"
        f"Current dependencies: {config['dependencies']}"
    )


def _update_http_app_path(
    config_path: pathlib.Path, config: Config, local_deps: LocalDeps
) -> None:
    """Update the HTTP app path to point to the correct location in the Docker container.

    Similar to _update_graph_paths, this ensures that if a custom app is specified via
    a local file path, that file is included in the Docker build context and its path
    is updated to point to the correct location in the container.
    """
    if not (http_config := config.get("http")) or not (
        app_str := http_config.get("app")
    ):
        return

    module_str, _, attr_str = app_str.partition(":")
    if not module_str or not attr_str:
        message = (
            'Import string "{import_str}" must be in format "<module>:<attribute>".'
        )
        raise ValueError(message.format(import_str=app_str))

    # Check if it's a file path
    if "/" in module_str or "\\" in module_str:
        # Resolve the local path properly on the current OS
        resolved = (config_path.parent / module_str).resolve()
        if not resolved.exists():
            raise FileNotFoundError(f"Could not find HTTP app module: {resolved}")
        elif not resolved.is_file():
            raise IsADirectoryError(f"HTTP app module must be a file: {resolved}")
        else:
            for path in local_deps.real_pkgs:
                if resolved.is_relative_to(path):
                    container_path = (
                        pathlib.Path("/deps") / path.name / resolved.relative_to(path)
                    )
                    module_str = container_path.as_posix()
                    break
            else:
                for faux_pkg, (_, destpath) in local_deps.faux_pkgs.items():
                    if resolved.is_relative_to(faux_pkg):
                        container_subpath = resolved.relative_to(faux_pkg)
                        # Construct the final path, ensuring POSIX style
                        module_str = f"{destpath}/{container_subpath.as_posix()}"
                        break
                else:
                    raise ValueError(
                        f"HTTP app module '{app_str}' not found in 'dependencies' list. "
                        "Add its containing package to 'dependencies' list."
                    )
        # update the config
        http_config["app"] = f"{module_str}:{attr_str}"


def _get_node_pm_install_cmd(config_path: pathlib.Path, config: Config) -> str:
    def test_file(file_name):
        full_path = config_path.parent / file_name
        try:
            return full_path.is_file()
        except OSError:
            return False

    # inspired by `package-manager-detector`
    def get_pkg_manager_name():
        try:
            with open(config_path.parent / "package.json") as f:
                pkg = json.load(f)

                if (pkg_manager_name := pkg.get("packageManager")) and isinstance(
                    pkg_manager_name, str
                ):
                    return pkg_manager_name.lstrip("^").split("@")[0]

                if (
                    dev_engine_name := (
                        (pkg.get("devEngines") or {}).get("packageManager") or {}
                    ).get("name")
                ) and isinstance(dev_engine_name, str):
                    return dev_engine_name

                return None
        except Exception:
            return None

    npm, yarn, pnpm, bun = [
        test_file("package-lock.json"),
        test_file("yarn.lock"),
        test_file("pnpm-lock.yaml"),
        test_file("bun.lockb"),
    ]

    if yarn:
        install_cmd = "yarn install --frozen-lockfile"
    elif pnpm:
        install_cmd = "pnpm i --frozen-lockfile"
    elif npm:
        install_cmd = "npm ci"
    elif bun:
        install_cmd = "bun i"
    else:
        pkg_manager_name = get_pkg_manager_name()

        if pkg_manager_name == "yarn":
            install_cmd = "yarn install"
        elif pkg_manager_name == "pnpm":
            install_cmd = "pnpm i"
        elif pkg_manager_name == "bun":
            install_cmd = "bun i"
        else:
            install_cmd = "npm i"

    return install_cmd


semver_pattern = re.compile(r":(\d+(?:\.\d+)?(?:\.\d+)?)(?:-|$)")


def _image_supports_uv(base_image: str) -> bool:
    if base_image == "langchain/langgraph-trial":
        return False
    match = semver_pattern.search(base_image)
    if not match:
        # Default image (langchain/langgraph-api) supports it.
        return True

    version_str = match.group(1)
    version = tuple(map(int, version_str.split(".")))
    min_uv = (0, 2, 47)
    return version >= min_uv


def get_build_tools_to_uninstall(config: Config) -> tuple[str]:
    keep_pkg_tools = config.get("keep_pkg_tools")
    if not keep_pkg_tools:
        return _BUILD_TOOLS
    if keep_pkg_tools is True:
        return ()
    expected = _BUILD_TOOLS
    if isinstance(keep_pkg_tools, list):
        for tool in keep_pkg_tools:
            if tool not in expected:
                raise ValueError(
                    f"Invalid build tool to uninstall: {tool}. Expected one of {expected}"
                )
        return tuple(sorted(set(_BUILD_TOOLS) - set(keep_pkg_tools)))
    else:
        raise ValueError(
            f"Invalid value for keep_pkg_tools: {keep_pkg_tools}."
            " Expected True or a list containing any of {expected}."
        )


def python_config_to_docker(
    config_path: pathlib.Path,
    config: Config,
    base_image: str,
    api_version: str | None = None,
) -> tuple[str, dict[str, str]]:
    """Generate a Dockerfile from the configuration."""
    pip_installer = config.get("pip_installer", "auto")
    build_tools_to_uninstall = get_build_tools_to_uninstall(config)
    if pip_installer == "auto":
        if _image_supports_uv(base_image):
            pip_installer = "uv"
        else:
            pip_installer = "pip"
    if pip_installer == "uv":
        install_cmd = "uv pip install --system"
    elif pip_installer == "pip":
        install_cmd = "pip install"
    else:
        raise ValueError(f"Invalid pip_installer: {pip_installer}")

    # configure pip
    local_reqs_pip_install = f"PYTHONDONTWRITEBYTECODE=1 {install_cmd} --no-cache-dir -c /api/constraints.txt"
    global_reqs_pip_install = f"PYTHONDONTWRITEBYTECODE=1 {install_cmd} --no-cache-dir -c /api/constraints.txt"
    if config.get("pip_config_file"):
        local_reqs_pip_install = (
            f"PIP_CONFIG_FILE=/pipconfig.txt {local_reqs_pip_install}"
        )
        global_reqs_pip_install = (
            f"PIP_CONFIG_FILE=/pipconfig.txt {global_reqs_pip_install}"
        )
    pip_config_file_str = (
        f"ADD {config['pip_config_file']} /pipconfig.txt"
        if config.get("pip_config_file")
        else ""
    )

    # collect dependencies
    pypi_deps = [dep for dep in config["dependencies"] if not dep.startswith(".")]
    local_deps = _assemble_local_deps(config_path, config)
    # Rewrite graph paths, so they point to the correct location in the Docker container
    _update_graph_paths(config_path, config, local_deps)
    # Rewrite auth path, so it points to the correct location in the Docker container
    _update_auth_path(config_path, config, local_deps)
    # Rewrite HTTP app path, so it points to the correct location in the Docker container
    _update_http_app_path(config_path, config, local_deps)

    pip_pkgs_str = (
        f"RUN {local_reqs_pip_install} {' '.join(pypi_deps)}" if pypi_deps else ""
    )
    if local_deps.pip_reqs:
        pip_reqs_str = os.linesep.join(
            (
                f"COPY --from=outer-{reqpath.name} requirements.txt {destpath}"
                if reqpath.parent in local_deps.additional_contexts
                else f"ADD {reqpath.relative_to(config_path.parent)} {destpath}"
            )
            for reqpath, destpath in local_deps.pip_reqs
        )
        pip_reqs_str += f"{os.linesep}RUN {local_reqs_pip_install} {' '.join('-r ' + r for _, r in local_deps.pip_reqs)}"
        pip_reqs_str = f"""# -- Installing local requirements --
{pip_reqs_str}
# -- End of local requirements install --"""

    else:
        pip_reqs_str = ""

    # https://setuptools.pypa.io/en/latest/userguide/datafiles.html#package-data
    # https://til.simonwillison.net/python/pyproject
    faux_pkgs_str = f"{os.linesep}{os.linesep}".join(
        (
            f"""# -- Adding non-package dependency {fullpath.name} --
COPY --from=outer-{fullpath.name} . {destpath}"""
            if fullpath in local_deps.additional_contexts
            else f"""# -- Adding non-package dependency {fullpath.name} --
ADD {relpath} {destpath}"""
        )
        + f"""
RUN set -ex && \\
    for line in '[project]' \\
                'name = "{fullpath.name}"' \\
                'version = "0.1"' \\
                '[tool.setuptools.package-data]' \\
                '"*" = ["**/*"]' \\
                '[build-system]' \\
                'requires = ["setuptools>=61"]' \\
                'build-backend = "setuptools.build_meta"'; do \\
        echo "$line" >> /deps/outer-{fullpath.name}/pyproject.toml; \\
    done
# -- End of non-package dependency {fullpath.name} --"""
        for fullpath, (relpath, destpath) in local_deps.faux_pkgs.items()
    )

    local_pkgs_str = os.linesep.join(
        (
            f"""# -- Adding local package {relpath} --
COPY --from={name} . /deps/{name}
# -- End of local package {relpath} --"""
            if fullpath in local_deps.additional_contexts
            else f"""# -- Adding local package {relpath} --
ADD {relpath} /deps/{name}
# -- End of local package {relpath} --"""
        )
        for fullpath, (relpath, name) in local_deps.real_pkgs.items()
    )

    install_node_str: str = (
        "RUN /storage/install-node.sh"
        if (config.get("ui") or config.get("node_version")) and local_deps.working_dir
        else ""
    )

    installs = f"{os.linesep}{os.linesep}".join(
        filter(
            None,
            [
                install_node_str,
                pip_config_file_str,
                pip_pkgs_str,
                pip_reqs_str,
                local_pkgs_str,
                faux_pkgs_str,
            ],
        )
    )

    env_vars = []

    if (store_config := config.get("store")) is not None:
        env_vars.append(f"ENV LANGGRAPH_STORE='{json.dumps(store_config)}'")

    if (auth_config := config.get("auth")) is not None:
        env_vars.append(f"ENV LANGGRAPH_AUTH='{json.dumps(auth_config)}'")

    if (http_config := config.get("http")) is not None:
        env_vars.append(f"ENV LANGGRAPH_HTTP='{json.dumps(http_config)}'")

    if (checkpointer_config := config.get("checkpointer")) is not None:
        env_vars.append(
            f"ENV LANGGRAPH_CHECKPOINTER='{json.dumps(checkpointer_config)}'"
        )

    if (ui := config.get("ui")) is not None:
        env_vars.append(f"ENV LANGGRAPH_UI='{json.dumps(ui)}'")

    if (ui_config := config.get("ui_config")) is not None:
        env_vars.append(f"ENV LANGGRAPH_UI_CONFIG='{json.dumps(ui_config)}'")

    env_vars.append(f"ENV LANGSERVE_GRAPHS='{json.dumps(config['graphs'])}'")

    js_inst_str: str = ""
    if (config.get("ui") or config.get("node_version")) and local_deps.working_dir:
        js_inst_str = os.linesep.join(
            [
                "# -- Installing JS dependencies --",
                f"ENV NODE_VERSION={config.get('node_version') or DEFAULT_NODE_VERSION}",
                f"RUN cd {local_deps.working_dir} && {_get_node_pm_install_cmd(config_path, config)} && tsx /api/langgraph_api/js/build.mts",
                "# -- End of JS dependencies install --",
            ]
        )
    image_str = docker_tag(config, base_image, api_version)

    # Prepare docker file contents
    docker_file_contents = []

    # Add syntax directive if we have additional contexts (requires BuildKit frontend.contexts capability)
    if local_deps.additional_contexts:
        docker_file_contents.extend(
            [
                "# syntax=docker/dockerfile:1.4",
                "",
            ]
        )

    # Add main dockerfile content
    docker_file_contents.extend(
        [
            f"FROM {image_str}",
            "",
            os.linesep.join(config["dockerfile_lines"]),
            "",
            installs,
            "",
            "# -- Installing all local dependencies --",
            f"""RUN for dep in /deps/*; do \
            echo "Installing $dep"; \
            if [ -d "$dep" ]; then \
                echo "Installing $dep"; \
                (cd "$dep" && {global_reqs_pip_install} -e .); \
            fi; \
        done""",
            "# -- End of local dependencies install --",
            os.linesep.join(env_vars),
            "",
            js_inst_str,
            "",
            # Add pip cleanup after all installations are complete
            _get_pip_cleanup_lines(
                install_cmd=install_cmd,
                to_uninstall=build_tools_to_uninstall,
                pip_installer=pip_installer,
            ),
            "",
            f"WORKDIR {local_deps.working_dir}" if local_deps.working_dir else "",
        ]
    )

    additional_contexts: dict[str, str] = {}
    for p in local_deps.additional_contexts:
        if p in local_deps.real_pkgs:
            name = local_deps.real_pkgs[p][1]
        elif p in local_deps.faux_pkgs:
            name = f"outer-{p.name}"
        else:
            raise RuntimeError(f"Unknown additional context: {p}")
        additional_contexts[name] = str(p)

    return os.linesep.join(docker_file_contents), additional_contexts


def node_config_to_docker(
    config_path: pathlib.Path,
    config: Config,
    base_image: str,
    api_version: str | None = None,
    install_command: str | None = None,
    build_command: str | None = None,
    build_context: str | None = None,
) -> tuple[str, dict[str, str]]:
    # Calculate paths for monorepo support
    if build_context:
        relative_workdir = _calculate_relative_workdir(config_path, build_context)
        container_name = pathlib.Path(build_context).name
        if relative_workdir:
            faux_path = f"/deps/{container_name}/{relative_workdir}"
        else:
            faux_path = f"/deps/{container_name}"
    else:
        # Backward compatibility: use the original behavior
        faux_path = f"/deps/{config_path.parent.name}"

    # Use custom install command or auto-detect
    if install_command:
        install_cmd = install_command
    else:
        install_cmd = _get_node_pm_install_cmd(config_path, config)

    image_str = docker_tag(config, base_image, api_version)

    env_vars: list[str] = []

    if (store_config := config.get("store")) is not None:
        env_vars.append(f"ENV LANGGRAPH_STORE='{json.dumps(store_config)}'")

    if (auth_config := config.get("auth")) is not None:
        env_vars.append(f"ENV LANGGRAPH_AUTH='{json.dumps(auth_config)}'")

    if (http_config := config.get("http")) is not None:
        env_vars.append(f"ENV LANGGRAPH_HTTP='{json.dumps(http_config)}'")

    if (checkpointer_config := config.get("checkpointer")) is not None:
        env_vars.append(
            f"ENV LANGGRAPH_CHECKPOINTER='{json.dumps(checkpointer_config)}'"
        )

    if ui := config.get("ui"):
        env_vars.append(f"ENV LANGGRAPH_UI='{json.dumps(ui)}'")

    if ui_config := config.get("ui_config"):
        env_vars.append(f"ENV LANGGRAPH_UI_CONFIG='{json.dumps(ui_config)}'")

    env_vars.append(f"ENV LANGSERVE_GRAPHS='{json.dumps(config['graphs'])}'")

    # For monorepo support, we need to handle install and build commands differently
    if build_context:
        # Monorepo case: install from root, build from config directory
        container_root = f"/deps/{pathlib.Path(build_context).name}"
        install_step = f"RUN cd {container_root} && {install_cmd}"

        if build_command:
            build_step = f"RUN cd {faux_path} && {build_command}"
        else:
            build_step = 'RUN (test ! -f /api/langgraph_api/js/build.mts && echo "Prebuild script not found, skipping") || tsx /api/langgraph_api/js/build.mts'
    else:
        # Original behavior: everything happens in the same directory
        install_step = f"RUN cd {faux_path} && {install_cmd}"
        build_step = 'RUN (test ! -f /api/langgraph_api/js/build.mts && echo "Prebuild script not found, skipping") || tsx /api/langgraph_api/js/build.mts'

    docker_file_contents = [
        f"FROM {image_str}",
        "",
        os.linesep.join(config["dockerfile_lines"]),
        "",
        f"ADD . {faux_path if not build_context else container_root}",
        "",
        install_step,
        "",
        os.linesep.join(env_vars),
        "",
        f"WORKDIR {faux_path}",
        "",
        build_step,
    ]

    return os.linesep.join(docker_file_contents), {}


def default_base_image(config: Config) -> str:
    if config.get("base_image"):
        return config["base_image"]
    if config.get("node_version") and not config.get("python_version"):
        return "langchain/langgraphjs-api"
    return "langchain/langgraph-api"


def docker_tag(
    config: Config,
    base_image: str | None = None,
    api_version: str | None = None,
) -> str:
    api_version = api_version or config.get("api_version")
    base_image = base_image or default_base_image(config)

    image_distro = config.get("image_distro")
    distro_tag = "" if image_distro == DEFAULT_IMAGE_DISTRO else f"-{image_distro}"

    if config.get("_INTERNAL_docker_tag"):
        return f"{base_image}:{config['_INTERNAL_docker_tag']}"

    # Build the standard tag format
    language, version = None, None
    if config.get("node_version") and not config.get("python_version"):
        language, version = "node", config["node_version"]
    else:
        language, version = "py", config["python_version"]

    version_distro_tag = f"{version}{distro_tag}"

    # Prepend API version if provided
    if api_version:
        full_tag = f"{api_version}-{language}{version_distro_tag}"
    elif "/langgraph-server" in base_image and version_distro_tag not in base_image:
        return f"{base_image}-{language}{version_distro_tag}"
    else:
        full_tag = version_distro_tag

    return f"{base_image}:{full_tag}"


def _calculate_relative_workdir(config_path: pathlib.Path, build_context: str) -> str:
    """Calculate the relative path from build context to langgraph.json directory."""
    config_dir = config_path.parent.resolve()
    build_context_path = pathlib.Path(build_context).resolve()

    try:
        relative_path = config_dir.relative_to(build_context_path)
        return str(relative_path) if str(relative_path) != "." else ""
    except ValueError as _:
        raise ValueError(
            f"Configuration file {config_path} is not under the build context {build_context}. "
            f"Please run the command from a directory that contains your langgraph.json file, "
        ) from None


def config_to_docker(
    config_path: pathlib.Path,
    config: Config,
    base_image: str | None = None,
    api_version: str | None = None,
    install_command: str | None = None,
    build_command: str | None = None,
    build_context: str | None = None,
) -> tuple[str, dict[str, str]]:
    base_image = base_image or default_base_image(config)

    if config.get("node_version") and not config.get("python_version"):
        return node_config_to_docker(
            config_path,
            config,
            base_image,
            api_version,
            install_command,
            build_command,
            build_context,
        )

    return python_config_to_docker(config_path, config, base_image, api_version)


def config_to_compose(
    config_path: pathlib.Path,
    config: Config,
    base_image: str | None = None,
    api_version: str | None = None,
    image: str | None = None,
    watch: bool = False,
) -> str:
    base_image = base_image or default_base_image(config)

    env_vars = config["env"].items() if isinstance(config["env"], dict) else {}
    env_vars_str = "\n".join(f'            {k}: "{v}"' for k, v in env_vars)
    env_file_str = (
        f"env_file: {config['env']}" if isinstance(config["env"], str) else ""
    )
    if watch:
        dependencies = config.get("dependencies") or ["."]
        watch_paths = [config_path.name] + [
            dep for dep in dependencies if dep.startswith(".")
        ]
        watch_actions = "\n".join(
            f"""- path: {path}
  action: rebuild"""
            for path in watch_paths
        )
        watch_str = f"""
        develop:
            watch:
{textwrap.indent(watch_actions, "                ")}
"""
    else:
        watch_str = ""
    if image:
        return f"""
{textwrap.indent(env_vars_str, "            ")}
        {env_file_str}
        {watch_str}
"""

    else:
        dockerfile, additional_contexts = config_to_docker(
            config_path, config, base_image, api_version
        )

        additional_contexts_str = "\n".join(
            f"                - {name}: {path}"
            for name, path in additional_contexts.items()
        )
        if additional_contexts_str:
            additional_contexts_str = f"""
            additional_contexts:
{additional_contexts_str}"""

        return f"""
{textwrap.indent(env_vars_str, "            ")}
        {env_file_str}
        pull_policy: build
        build:
            context: .{additional_contexts_str}
            dockerfile_inline: |
{textwrap.indent(dockerfile, "                ")}
        {watch_str}
"""

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/constants.py
```py
DEFAULT_CONFIG = "langgraph.json"
DEFAULT_PORT = 8123

# analytics
SUPABASE_PUBLIC_API_KEY = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6Imt6cmxwcG9qaW5wY3l5YWlweG5iIiwicm9sZSI6ImFub24iLCJpYXQiOjE3MTkyNTc1NzksImV4cCI6MjAzNDgzMzU3OX0.kkVOlLz3BxemA5nP-vat3K4qRtrDuO4SwZSR_htcX9c"
SUPABASE_URL = "https://kzrlppojinpcyyaipxnb.supabase.co"

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/docker.py
```py
import json
import pathlib
import shutil
from typing import Literal, NamedTuple

import click.exceptions

from langgraph_cli.exec import subp_exec

ROOT = pathlib.Path(__file__).parent.resolve()
DEFAULT_POSTGRES_URI = (
    "postgres://postgres:postgres@langgraph-postgres:5432/postgres?sslmode=disable"
)


class Version(NamedTuple):
    major: int
    minor: int
    patch: int


DockerComposeType = Literal["plugin", "standalone"]


class DockerCapabilities(NamedTuple):
    version_docker: Version
    version_compose: Version
    healthcheck_start_interval: bool
    compose_type: DockerComposeType = "plugin"


def _parse_version(version: str) -> Version:
    parts = version.split(".", 2)
    if len(parts) == 1:
        major = parts[0]
        minor = "0"
        patch = "0"
    elif len(parts) == 2:
        major, minor = parts
        patch = "0"
    else:
        major, minor, patch = parts
    return Version(
        int(major.lstrip("v")), int(minor), int(patch.split("-")[0].split("+")[0])
    )


def check_capabilities(runner) -> DockerCapabilities:
    # check docker available
    if shutil.which("docker") is None:
        raise click.UsageError("Docker not installed") from None

    try:
        stdout, _ = runner.run(
            subp_exec("docker", "info", "-f", "{{json .}}", collect=True)
        )
        info = json.loads(stdout)
    except (click.exceptions.Exit, json.JSONDecodeError):
        raise click.UsageError("Docker not installed or not running") from None

    if not info["ServerVersion"]:
        raise click.UsageError("Docker not running") from None

    compose_type: DockerComposeType
    try:
        compose = next(
            p for p in info["ClientInfo"]["Plugins"] if p["Name"] == "compose"
        )
        compose_version_str = compose["Version"]
        compose_type = "plugin"
    except (KeyError, StopIteration):
        if shutil.which("docker-compose") is None:
            raise click.UsageError("Docker Compose not installed") from None

        compose_version_str, _ = runner.run(
            subp_exec("docker-compose", "--version", "--short", collect=True)
        )
        compose_type = "standalone"

    # parse versions
    docker_version = _parse_version(info["ServerVersion"])
    compose_version = _parse_version(compose_version_str)

    # check capabilities
    return DockerCapabilities(
        version_docker=docker_version,
        version_compose=compose_version,
        healthcheck_start_interval=docker_version >= Version(25, 0, 0),
        compose_type=compose_type,
    )


def debugger_compose(*, port: int | None = None, base_url: str | None = None) -> dict:
    if port is None:
        return ""

    config = {
        "langgraph-debugger": {
            "image": "langchain/langgraph-debugger",
            "restart": "on-failure",
            "depends_on": {
                "langgraph-postgres": {"condition": "service_healthy"},
            },
            "ports": [f'"{port}:3968"'],
        }
    }

    if base_url:
        config["langgraph-debugger"]["environment"] = {
            "VITE_STUDIO_LOCAL_GRAPH_URL": base_url
        }

    return config


# Function to convert dictionary to YAML
def dict_to_yaml(d: dict, *, indent: int = 0) -> str:
    """Convert a dictionary to a YAML string."""
    yaml_str = ""

    for idx, (key, value) in enumerate(d.items()):
        # Format things in a visually appealing way
        # Use an extra newline for top-level keys only
        if idx >= 1 and indent < 2:
            yaml_str += "\n"
        space = "    " * indent
        if isinstance(value, dict):
            yaml_str += f"{space}{key}:\n" + dict_to_yaml(value, indent=indent + 1)
        elif isinstance(value, list):
            yaml_str += f"{space}{key}:\n"
            for item in value:
                yaml_str += f"{space}    - {item}\n"
        else:
            yaml_str += f"{space}{key}: {value}\n"
    return yaml_str


def compose_as_dict(
    capabilities: DockerCapabilities,
    *,
    port: int,
    debugger_port: int | None = None,
    debugger_base_url: str | None = None,
    # postgres://user:password@host:port/database?option=value
    postgres_uri: str | None = None,
    # If you are running against an already-built image, you can pass it here
    image: str | None = None,
    # Base image to use for the LangGraph API server
    base_image: str | None = None,
    # API version of the base image
    api_version: str | None = None,
) -> dict:
    """Create a docker compose file as a dictionary in YML style."""
    if postgres_uri is None:
        include_db = True
        postgres_uri = DEFAULT_POSTGRES_URI
    else:
        include_db = False

    # The services below are defined in a non-intuitive order to match
    # the existing unit tests for this function.
    # It's fine to re-order just requires updating the unit tests, so it should
    # be done with caution.

    # Define the Redis service first as per the test order
    services = {
        "langgraph-redis": {
            "image": "redis:6",
            "healthcheck": {
                "test": "redis-cli ping",
                "interval": "5s",
                "timeout": "1s",
                "retries": 5,
            },
        }
    }

    # Add Postgres service before langgraph-api if it is needed
    if include_db:
        services["langgraph-postgres"] = {
            "image": "pgvector/pgvector:pg16",
            "ports": ['"5433:5432"'],
            "environment": {
                "POSTGRES_DB": "postgres",
                "POSTGRES_USER": "postgres",
                "POSTGRES_PASSWORD": "postgres",
            },
            "command": ["postgres", "-c", "shared_preload_libraries=vector"],
            "volumes": ["langgraph-data:/var/lib/postgresql/data"],
            "healthcheck": {
                "test": "pg_isready -U postgres",
                "start_period": "10s",
                "timeout": "1s",
                "retries": 5,
            },
        }
        if capabilities.healthcheck_start_interval:
            services["langgraph-postgres"]["healthcheck"]["interval"] = "60s"
            services["langgraph-postgres"]["healthcheck"]["start_interval"] = "1s"
        else:
            services["langgraph-postgres"]["healthcheck"]["interval"] = "5s"

    # Add optional debugger service if debugger_port is specified
    if debugger_port:
        services["langgraph-debugger"] = debugger_compose(
            port=debugger_port, base_url=debugger_base_url
        )["langgraph-debugger"]

    # Add langgraph-api service
    services["langgraph-api"] = {
        "ports": [f'"{port}:8000"'],
        "depends_on": {
            "langgraph-redis": {"condition": "service_healthy"},
        },
        "environment": {
            "REDIS_URI": "redis://langgraph-redis:6379",
            "POSTGRES_URI": postgres_uri,
        },
    }
    if image:
        services["langgraph-api"]["image"] = image

    # If Postgres is included, add it to the dependencies of langgraph-api
    if include_db:
        services["langgraph-api"]["depends_on"]["langgraph-postgres"] = {
            "condition": "service_healthy"
        }

    # Additional healthcheck for langgraph-api if required
    if capabilities.healthcheck_start_interval:
        services["langgraph-api"]["healthcheck"] = {
            "test": "python /api/healthcheck.py",
            "interval": "60s",
            "start_interval": "1s",
            "start_period": "10s",
        }

    # Final compose dictionary with volumes included if needed
    compose_dict = {}
    if include_db:
        compose_dict["volumes"] = {"langgraph-data": {"driver": "local"}}
    compose_dict["services"] = services

    return compose_dict


def compose(
    capabilities: DockerCapabilities,
    *,
    port: int,
    debugger_port: int | None = None,
    debugger_base_url: str | None = None,
    # postgres://user:password@host:port/database?option=value
    postgres_uri: str | None = None,
    image: str | None = None,
    base_image: str | None = None,
    api_version: str | None = None,
) -> str:
    """Create a docker compose file as a string."""
    compose_content = compose_as_dict(
        capabilities,
        port=port,
        debugger_port=debugger_port,
        debugger_base_url=debugger_base_url,
        postgres_uri=postgres_uri,
        image=image,
        base_image=base_image,
        api_version=api_version,
    )
    compose_str = dict_to_yaml(compose_content)
    return compose_str

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/exec.py
```py
import asyncio
import signal
import sys
from collections.abc import Callable
from contextlib import contextmanager
from typing import cast

import click.exceptions


@contextmanager
def Runner():
    if hasattr(asyncio, "Runner"):
        with asyncio.Runner() as runner:
            yield runner
    else:

        class _Runner:
            def __enter__(self):
                return self

            def __exit__(self, *args):
                pass

            def run(self, coro):
                return asyncio.run(coro)

        yield _Runner()


async def subp_exec(
    cmd: str,
    *args: str,
    input: str | None = None,
    wait: float | None = None,
    verbose: bool = False,
    collect: bool = False,
    on_stdout: Callable[[str], bool | None] | None = None,
) -> tuple[str | None, str | None]:
    if verbose:
        cmd_str = f"+ {cmd} {' '.join(map(str, args))}"
        if input:
            print(cmd_str, " <\n", "\n".join(filter(None, input.splitlines())), sep="")
        else:
            print(cmd_str)
    if wait:
        await asyncio.sleep(wait)

    try:
        proc = await asyncio.create_subprocess_exec(
            cmd,
            *args,
            stdin=asyncio.subprocess.PIPE if input else None,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )

        def signal_handler():
            # make sure process exists, then terminate it
            if proc.returncode is None:
                proc.terminate()

        original_sigint_handler = signal.getsignal(signal.SIGINT)
        if sys.platform == "win32":

            def handle_windows_signal(signum, frame):
                signal_handler()
                original_sigint_handler(signum, frame)

            signal.signal(signal.SIGINT, handle_windows_signal)
            # NOTE: we're not adding a handler for SIGTERM since it's ignored on Windows
        else:
            loop = asyncio.get_event_loop()
            loop.add_signal_handler(signal.SIGINT, signal_handler)
            loop.add_signal_handler(signal.SIGTERM, signal_handler)

        empty_fut: asyncio.Future = asyncio.Future()
        empty_fut.set_result(None)
        stdout, stderr, _ = await asyncio.gather(
            monitor_stream(
                cast(asyncio.StreamReader, proc.stdout),
                collect=True,
                display=verbose,
                on_line=on_stdout,
            ),
            monitor_stream(
                cast(asyncio.StreamReader, proc.stderr),
                collect=True,
                display=verbose,
            ),
            proc._feed_stdin(input.encode()) if input else empty_fut,  # type: ignore[attr-defined]
        )
        returncode = await proc.wait()
        if (
            returncode is not None
            and returncode != 0  # success
            and returncode != 130  # user interrupt
        ):
            sys.stdout.write(stdout.decode() if stdout else "")
            sys.stderr.write(stderr.decode() if stderr else "")
            raise click.exceptions.Exit(returncode)
        if collect:
            return (
                stdout.decode() if stdout else None,
                stderr.decode() if stderr else None,
            )
        else:
            return None, None
    finally:
        try:
            if proc.returncode is None:
                try:
                    proc.terminate()
                except (ProcessLookupError, KeyboardInterrupt):
                    pass

            if sys.platform == "win32":
                signal.signal(signal.SIGINT, original_sigint_handler)
            else:
                loop.remove_signal_handler(signal.SIGINT)
                loop.remove_signal_handler(signal.SIGTERM)
        except UnboundLocalError:
            pass


async def monitor_stream(
    stream: asyncio.StreamReader,
    collect: bool = False,
    display: bool = False,
    on_line: Callable[[str], bool | None] | None = None,
) -> bytearray | None:
    if collect:
        ba = bytearray()

    def handle(line: bytes, overrun: bool):
        nonlocal on_line
        nonlocal display

        if display:
            sys.stdout.buffer.write(line)
        if overrun:
            return
        if collect:
            ba.extend(line)
        if on_line:
            if on_line(line.decode()):
                on_line = None
                display = True

    """Adapted from asyncio.StreamReader.readline() to handle LimitOverrunError."""
    sep = b"\n"
    seplen = len(sep)
    while True:
        try:
            line = await stream.readuntil(sep)
            overrun = False
        except asyncio.IncompleteReadError as e:
            line = e.partial
            overrun = False
        except asyncio.LimitOverrunError as e:
            if stream._buffer.startswith(sep, e.consumed):
                line = stream._buffer[: e.consumed + seplen]
            else:
                line = stream._buffer.clear()
            overrun = True
            stream._maybe_resume_transport()
        await asyncio.to_thread(handle, line, overrun)
        if line == b"":
            break

    if collect:
        return ba
    else:
        return None

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/progress.py
```py
import sys
import threading
import time
from collections.abc import Callable


class Progress:
    delay: float = 0.1

    @staticmethod
    def spinning_cursor():
        while True:
            yield from "|/-\\"

    def __init__(self, *, message=""):
        self.message = message
        self.spinner_generator = self.spinning_cursor()

    def spinner_iteration(self):
        message = self.message
        sys.stdout.write(next(self.spinner_generator) + " " + message)
        sys.stdout.flush()
        time.sleep(self.delay)
        # clear the spinner and message
        sys.stdout.write(
            "\b" * (len(message) + 2)
            + " " * (len(message) + 2)
            + "\b" * (len(message) + 2)
        )
        sys.stdout.flush()

    def spinner_task(self):
        while self.message:
            message = self.message
            sys.stdout.write(next(self.spinner_generator) + " " + message)
            sys.stdout.flush()
            time.sleep(self.delay)
            # clear the spinner and message
            sys.stdout.write(
                "\b" * (len(message) + 2)
                + " " * (len(message) + 2)
                + "\b" * (len(message) + 2)
            )
            sys.stdout.flush()

    def __enter__(self) -> Callable[[str], None]:
        if sys.stdout.isatty():
            self.thread = threading.Thread(target=self.spinner_task)
            self.thread.start()

            def set_message(message):
                self.message = message
                if not message:
                    self.thread.join()

            return set_message
        else:

            def set_message(message):
                sys.stderr.write(message + "\n")
                sys.stderr.flush()

            return set_message

    def __exit__(self, exception, value, tb):
        if sys.stdout.isatty():
            self.message = ""
            try:
                self.thread.join()
            finally:
                del self.thread
            if exception is not None:
                return False

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/py.typed
```typed

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/schemas.py
```py
from typing import Any, Literal, TypedDict

Distros = Literal["debian", "wolfi", "bullseye", "bookworm"]
MiddlewareOrders = Literal["auth_first", "middleware_first"]


class TTLConfig(TypedDict, total=False):
    """Configuration for TTL (time-to-live) behavior in the store."""

    refresh_on_read: bool
    """Default behavior for refreshing TTLs on read operations (`GET` and `SEARCH`).
    
    If `True`, TTLs will be refreshed on read operations (get/search) by default.
    This can be overridden per-operation by explicitly setting `refresh_ttl`.
    Defaults to `True` if not configured.
    """
    default_ttl: float | None
    """Optional. Default TTL (time-to-live) in minutes for new items.
    
    If provided, all new items will have this TTL unless explicitly overridden.
    If omitted, items will have no TTL by default.
    """
    sweep_interval_minutes: int | None
    """Optional. Interval in minutes between TTL sweep iterations.
    
    If provided, the store will periodically delete expired items based on the TTL.
    If omitted, no automatic sweeping will occur.
    """


class IndexConfig(TypedDict, total=False):
    """Configuration for indexing documents for semantic search in the store.

    This governs how text is converted into embeddings and stored for vector-based lookups.
    """

    dims: int
    """Required. Dimensionality of the embedding vectors you will store.
    
    Must match the output dimension of your selected embedding model or custom embed function.
    If mismatched, you will likely encounter shape/size errors when inserting or querying vectors.
    
    Common embedding model output dimensions:
        - openai:text-embedding-3-large: 3072
        - openai:text-embedding-3-small: 1536
        - openai:text-embedding-ada-002: 1536
        - cohere:embed-english-v3.0: 1024
        - cohere:embed-english-light-v3.0: 384
        - cohere:embed-multilingual-v3.0: 1024
        - cohere:embed-multilingual-light-v3.0: 384
    """

    embed: str
    """Required. Identifier or reference to the embedding model or a custom embedding function.
    
    The format can vary:
      - "<provider>:<model_name>" for recognized providers (e.g., "openai:text-embedding-3-large")
      - "path/to/module.py:function_name" for your own local embedding function
      - "my_custom_embed" if it's a known alias in your system

     Examples:
        - "openai:text-embedding-3-large"
        - "cohere:embed-multilingual-v3.0"
        - "src/app.py:embeddings"
    
    Note: Must return embeddings of dimension `dims`.
    """

    fields: list[str] | None
    """Optional. List of JSON fields to extract before generating embeddings.
    
    Defaults to ["$"], which means the entire JSON object is embedded as one piece of text.
    If you provide multiple fields (e.g. ["title", "content"]), each is extracted and embedded separately,
    often saving token usage if you only care about certain parts of the data.
    
    Example:
        fields=["title", "abstract", "author.biography"]
    """


class StoreConfig(TypedDict, total=False):
    """Configuration for the built-in long-term memory store.

    This store can optionally perform semantic search. If you omit `index`,
    the store will just handle traditional (non-embedded) data without vector lookups.
    """

    index: IndexConfig | None
    """Optional. Defines the vector-based semantic search configuration.
    
    If provided, the store will:
      - Generate embeddings according to `index.embed`
      - Enforce the embedding dimension given by `index.dims`
      - Embed only specified JSON fields (if any) from `index.fields`
    
    If omitted, no vector index is initialized.
    """

    ttl: TTLConfig | None
    """Optional. Defines the TTL (time-to-live) behavior configuration.
    
    If provided, the store will apply TTL settings according to the configuration.
    If omitted, no TTL behavior is configured.
    """


class ThreadTTLConfig(TypedDict, total=False):
    """Configure a default TTL for checkpointed data within threads."""

    strategy: Literal["delete"]
    """Strategy to use for deleting checkpointed data.
    
    Choices:
      - "delete": Delete all checkpoints for a thread after TTL expires.
    """
    default_ttl: float | None
    """Default TTL (time-to-live) in minutes for checkpointed data."""
    sweep_interval_minutes: int | None
    """Interval in minutes between sweep iterations.
    If omitted, a default interval will be used (typically ~ 5 minutes)."""


class SerdeConfig(TypedDict, total=False):
    """Configuration for the built-in serde, which handles checkpointing of state.

    If omitted, no serde is set up (the object store will still be present, however)."""

    allowed_json_modules: list[list[str]] | bool | None
    """Optional. List of allowed python modules to de-serialize custom objects from.
    
    If provided, only the specified modules will be allowed to be deserialized.
    If omitted, no modules are allowed, and the object returned will simply be a json object OR
    a deserialized langchain object.
    
    Example:
    {...
        "serde": {
            "allowed_json_modules": [
                ["my_agent", "my_file", "SomeType"],
            ]
        }
    }

    If you set this to True, any module will be allowed to be deserialized.

    Example:
    {...
        "serde": {
            "allowed_json_modules": true
        }
    }
    
    """
    pickle_fallback: bool
    """Optional. Whether to allow pickling as a fallback for deserialization.
    
    If True, pickling will be allowed as a fallback for deserialization.
    If False, pickling will not be allowed as a fallback for deserialization.
    Defaults to True if not configured."""


class CheckpointerConfig(TypedDict, total=False):
    """Configuration for the built-in checkpointer, which handles checkpointing of state.

    If omitted, no checkpointer is set up (the object store will still be present, however).
    """

    ttl: ThreadTTLConfig | None
    """Optional. Defines the TTL (time-to-live) behavior configuration.
    
    If provided, the checkpointer will apply TTL settings according to the configuration.
    If omitted, no TTL behavior is configured.
    """
    serde: SerdeConfig | None
    """Optional. Defines the serde configuration.
    
    If provided, the checkpointer will apply serde settings according to the configuration.
    If omitted, no serde behavior is configured.

    This configuration requires server version 0.5 or later to take effect.
    """


class SecurityConfig(TypedDict, total=False):
    """Configuration for OpenAPI security definitions and requirements.

    Useful for specifying global or path-level authentication and authorization flows
    (e.g., OAuth2, API key headers, etc.).
    """

    securitySchemes: dict[str, dict[str, Any]]
    """Describe each security scheme recognized by your OpenAPI spec.
    
    Keys are scheme names (e.g. "OAuth2", "ApiKeyAuth") and values are their definitions.
    Example:
        {
            "OAuth2": {
                "type": "oauth2",
                "flows": {
                    "password": {
                        "tokenUrl": "/token",
                        "scopes": {"read": "Read data", "write": "Write data"}
                    }
                }
            }
        }
    """
    security: list[dict[str, list[str]]]
    """Global security requirements across all endpoints.
    
    Each element in the list maps a security scheme (e.g. "OAuth2") to a list of scopes (e.g. ["read", "write"]).
    Example:
        [
            {"OAuth2": ["read", "write"]},
            {"ApiKeyAuth": []}
        ]
    """
    # path => {method => security}
    paths: dict[str, dict[str, list[dict[str, list[str]]]]]
    """Path-specific security overrides.
    
    Keys are path templates (e.g., "/items/{item_id}"), mapping to:
      - Keys that are HTTP methods (e.g., "GET", "POST"),
      - Values are lists of security definitions (just like `security`) for that method.
    
    Example:
        {
            "/private_data": {
                "GET": [{"OAuth2": ["read"]}],
                "POST": [{"OAuth2": ["write"]}]
            }
        }
    """


class AuthConfig(TypedDict, total=False):
    """Configuration for custom authentication logic and how it integrates into the OpenAPI spec."""

    path: str
    """Required. Path to an instance of the Auth() class that implements custom authentication.
    
    Format: "path/to/file.py:my_auth"
    """
    disable_studio_auth: bool
    """Optional. Whether to disable LangSmith API-key authentication for requests originating the Studio. 
    
    Defaults to False, meaning that if a particular header is set, the server will verify the `x-api-key` header
    value is a valid API key for the deployment's workspace. If `True`, all requests will go through your custom
    authentication logic, regardless of origin of the request.
    """
    openapi: SecurityConfig
    """The security configuration to include in your server's OpenAPI spec.
    
    Example (OAuth2):
        {
            "securitySchemes": {
                "OAuth2": {
                    "type": "oauth2",
                    "flows": {
                        "password": {
                            "tokenUrl": "/token",
                            "scopes": {"me": "Read user info", "items": "Manage items"}
                        }
                    }
                }
            },
            "security": [
                {"OAuth2": ["me"]}
            ]
        }
    """


class CorsConfig(TypedDict, total=False):
    """Specifies Cross-Origin Resource Sharing (CORS) rules for your server.

    If omitted, defaults are typically very restrictive (often no cross-origin requests).
    Configure carefully if you want to allow usage from browsers hosted on other domains.
    """

    allow_origins: list[str]
    """Optional. List of allowed origins (e.g., "https://example.com").
    
    Default is often an empty list (no external origins). 
    Use "*" only if you trust all origins, as that bypasses most restrictions.
    """
    allow_methods: list[str]
    """Optional. HTTP methods permitted for cross-origin requests (e.g. ["GET", "POST"]).
    
    Default might be ["GET", "POST", "OPTIONS"] depending on your server framework.
    """
    allow_headers: list[str]
    """Optional. HTTP headers that can be used in cross-origin requests (e.g. ["Content-Type", "Authorization"])."""
    allow_credentials: bool
    """Optional. If `True`, cross-origin requests can include credentials (cookies, auth headers).
    
    Default False to avoid accidentally exposing secured endpoints to untrusted sites.
    """
    allow_origin_regex: str
    """Optional. A regex pattern for matching allowed origins, used if you have dynamic subdomains.
    
    Example: "^https://.*\\.mycompany\\.com$"
    """
    expose_headers: list[str]
    """Optional. List of headers that browsers are allowed to read from the response in cross-origin contexts."""
    max_age: int
    """Optional. How many seconds the browser may cache preflight responses.
    
    Default might be 600 (10 minutes). Larger values reduce preflight requests but can cause stale configurations.
    """


class ConfigurableHeaderConfig(TypedDict):
    """Customize which headers to include as configurable values in your runs.

    By default, omits x-api-key, x-tenant-id, and x-service-key.

    Exclusions (if provided) take precedence.

    Each value can be a raw string with an optional wildcard.
    """

    includes: list[str] | None
    """Headers to include (if not also matches against an 'exludes' pattern.

    Examples:
        - 'user-agent'
        - 'x-configurable-*'
    """
    excludes: list[str] | None
    """Headers to exclude. Applied before the 'includes' checks.

    Examples:
        - 'x-api-key'
        - '*key*'
        - '*token*'
    """


class HttpConfig(TypedDict, total=False):
    """Configuration for the built-in HTTP server that powers your deployment's routes and endpoints."""

    app: str
    """Optional. Import path to a custom Starlette/FastAPI application to mount.
    
    Format: "path/to/module.py:app_var"
    If provided, it can override or extend the default routes.
    """
    disable_assistants: bool
    """Optional. If `True`, /assistants routes are removed from the server.
    
    Default is False (meaning /assistants is enabled).
    """
    disable_threads: bool
    """Optional. If `True`, /threads routes are removed.
    
    Default is False.
    """
    disable_runs: bool
    """Optional. If `True`, /runs routes are removed.
    
    Default is False.
    """
    disable_store: bool
    """Optional. If `True`, /store routes are removed, disabling direct store interactions via HTTP.
    
    Default is False.
    """
    disable_mcp: bool
    """Optional. If `True`, /mcp routes are removed, disabling the MCP server.
    
    Default is False.
    """
    disable_meta: bool
    """Optional. Remove meta endpoints.
    
    Set to True to disable the following endpoints: /openapi.json, /info, /metrics, /docs.
    This will also make the /ok endpoint skip any DB or other checks, always returning {"ok": True}.
    
    Default is False.
    """
    cors: CorsConfig | None
    """Optional. Defines CORS restrictions. If omitted, no special rules are set and 
    cross-origin behavior depends on default server settings.
    """
    configurable_headers: ConfigurableHeaderConfig | None
    """Optional. Defines how headers are treated for a run's configuration.

    You can include or exclude headers as configurable values to condition your
    agent's behavior or permissions on a request's headers."""
    logging_headers: ConfigurableHeaderConfig | None
    """Optional. Defines which headers are excluded from logging."""
    middleware_order: MiddlewareOrders | None
    """Optional. Defines the order in which to apply server customizations.

    Choices:
      - "auth_first": Authentication hooks (custom or default) are evaluated
      before custom middleware.
      - "middleware_first": Custom middleware is evaluated
      before authentication hooks (custom or default).

    Default is `middleware_first`.
    """
    enable_custom_route_auth: bool
    """Optional. If `True`, authentication is enabled for custom routes,
    not just the routes that are protected by default.
    (Routes protected by default include /assistants, /threads, and /runs).

    Default is False. This flag only affects authentication behavior
    if `app` is provided and contains custom routes.
    """


class Config(TypedDict, total=False):
    """Top-level config for langgraph-cli or similar deployment tooling."""

    python_version: str
    """Optional. Python version in 'major.minor' format (e.g. '3.11'). 
    Must be at least 3.11 or greater for this deployment to function properly.
    """

    node_version: str | None
    """Optional. Node.js version as a major version (e.g. '20'), if your deployment needs Node.
    Must be >= 20 if provided.
    """

    api_version: str | None
    """Optional. Which semantic version of the LangGraph API server to use.
    
    Defaults to latest. Check the
    [changelog](https://docs.langchain.com/langgraph-platform/langgraph-server-changelog)
    for more information."""

    _INTERNAL_docker_tag: str | None
    """Optional. Internal use only.
    """

    base_image: str | None
    """Optional. Base image to use for the LangGraph API server.
    
    Defaults to langchain/langgraph-api or langchain/langgraphjs-api."""

    image_distro: Distros | None
    """Optional. Linux distribution for the base image.
    
    Must be one of 'wolfi', 'debian', 'bullseye', or 'bookworm'.
    If omitted, defaults to 'debian' ('latest').
    """

    pip_config_file: str | None
    """Optional. Path to a pip config file (e.g., "/etc/pip.conf" or "pip.ini") for controlling
    package installation (custom indices, credentials, etc.).
    
    Only relevant if Python dependencies are installed via pip. If omitted, default pip settings are used.
    """

    pip_installer: str | None
    """Optional. Python package installer to use ('auto', 'pip', 'uv').
    
    - 'auto' (default): Use uv for supported base images, otherwise pip
    - 'pip': Force use of pip regardless of base image support
    - 'uv': Force use of uv (will fail if base image doesn't support it)
    """

    dockerfile_lines: list[str]
    """Optional. Additional Docker instructions that will be appended to your base Dockerfile.
    
    Useful for installing OS packages, setting environment variables, etc. 
    Example:
        dockerfile_lines=[
            "RUN apt-get update && apt-get install -y libmagic-dev",
            "ENV MY_CUSTOM_VAR=hello_world"
        ]
    """

    dependencies: list[str]
    """List of Python dependencies to install, either from PyPI or local paths.
    
    Examples:
      - "." or "./src" if you have a local Python package
      - str (aka "anthropic") for a PyPI package
      - "git+https://github.com/org/repo.git@main" for a Git-based package
    Defaults to an empty list, meaning no additional packages installed beyond your base environment.
    """

    graphs: dict[str, str]
    """Optional. Named definitions of graphs, each pointing to a Python object.

    
    Graphs can be StateGraph, @entrypoint, or any other Pregel object OR they can point to (async) context
    managers that accept a single configuration argument (of type RunnableConfig) and return a pregel object
    (instance of Stategraph, etc.).
    
    Keys are graph names, values are "path/to/file.py:object_name".
    Example:
        {
            "mygraph": "graphs/my_graph.py:graph_definition",
            "anothergraph": "graphs/another.py:get_graph"
        }
    """

    env: dict[str, str] | str
    """Optional. Environment variables to set for your deployment.
    
    - If given as a dict, keys are variable names and values are their values.
    - If given as a string, it must be a path to a file containing lines in KEY=VALUE format.
    
    Example as a dict:
        env={"API_TOKEN": "abc123", "DEBUG": "true"}
    Example as a file path:
        env=".env"
    """

    store: StoreConfig | None
    """Optional. Configuration for the built-in long-term memory store, including semantic search indexing.
    
    If omitted, no vector index is set up (the object store will still be present, however).
    """

    checkpointer: CheckpointerConfig | None
    """Optional. Configuration for the built-in checkpointer, which handles checkpointing of state.
    
    If omitted, no checkpointer is set up (the object store will still be present, however).
    """

    auth: AuthConfig | None
    """Optional. Custom authentication config, including the path to your Python auth logic and 
    the OpenAPI security definitions it uses.
    """

    http: HttpConfig | None
    """Optional. Configuration for the built-in HTTP server, controlling which custom routes are exposed
    and how cross-origin requests are handled.
    """

    ui: dict[str, str] | None
    """Optional. Named definitions of UI components emitted by the agent, each pointing to a JS/TS file.
    """

    keep_pkg_tools: bool | list[str] | None
    """Optional. Control whether to retain Python packaging tools in the final image.
    
    Allowed tools are: "pip", "setuptools", "wheel".
    You can also set to true to include all packaging tools.
    """


__all__ = [
    "Config",
    "StoreConfig",
    "CheckpointerConfig",
    "AuthConfig",
    "HttpConfig",
    "MiddlewareOrders",
    "Distros",
    "TTLConfig",
    "IndexConfig",
]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/templates.py
```py
import os
import shutil
import sys
from io import BytesIO
from urllib import error, request
from zipfile import ZipFile

import click

TEMPLATES: dict[str, dict[str, str]] = {
    "New LangGraph Project": {
        "description": "A simple, minimal chatbot with memory.",
        "python": "https://github.com/langchain-ai/new-langgraph-project/archive/refs/heads/main.zip",
        "js": "https://github.com/langchain-ai/new-langgraphjs-project/archive/refs/heads/main.zip",
    },
    "ReAct Agent": {
        "description": "A simple agent that can be flexibly extended to many tools.",
        "python": "https://github.com/langchain-ai/react-agent/archive/refs/heads/main.zip",
        "js": "https://github.com/langchain-ai/react-agent-js/archive/refs/heads/main.zip",
    },
    "Memory Agent": {
        "description": "A ReAct-style agent with an additional tool to store memories for use across conversational threads.",
        "python": "https://github.com/langchain-ai/memory-agent/archive/refs/heads/main.zip",
        "js": "https://github.com/langchain-ai/memory-agent-js/archive/refs/heads/main.zip",
    },
    "Retrieval Agent": {
        "description": "An agent that includes a retrieval-based question-answering system.",
        "python": "https://github.com/langchain-ai/retrieval-agent-template/archive/refs/heads/main.zip",
        "js": "https://github.com/langchain-ai/retrieval-agent-template-js/archive/refs/heads/main.zip",
    },
    "Data-enrichment Agent": {
        "description": "An agent that performs web searches and organizes its findings into a structured format.",
        "python": "https://github.com/langchain-ai/data-enrichment/archive/refs/heads/main.zip",
        "js": "https://github.com/langchain-ai/data-enrichment-js/archive/refs/heads/main.zip",
    },
}

# Generate TEMPLATE_IDS programmatically
TEMPLATE_ID_TO_CONFIG = {
    f"{name.lower().replace(' ', '-')}-{lang}": (name, lang, url)
    for name, versions in TEMPLATES.items()
    for lang, url in versions.items()
    if lang in {"python", "js"}
}

TEMPLATE_IDS = list(TEMPLATE_ID_TO_CONFIG.keys())

TEMPLATE_HELP_STRING = (
    "The name of the template to use. Available options:\n"
    + "\n".join(f"{id_}" for id_ in TEMPLATE_ID_TO_CONFIG)
)


def _choose_template() -> str:
    """Presents a list of templates to the user and prompts them to select one.

    Returns:
        str: The URL of the selected template.
    """
    click.secho("🌟 Please select a template:", bold=True, fg="yellow")
    for idx, (template_name, template_info) in enumerate(TEMPLATES.items(), 1):
        click.secho(f"{idx}. ", nl=False, fg="cyan")
        click.secho(template_name, fg="cyan", nl=False)
        click.secho(f" - {template_info['description']}", fg="white")

    # Get the template choice from the user, defaulting to the first template if blank
    template_choice: int | None = click.prompt(
        "Enter the number of your template choice (default is 1)",
        type=int,
        default=1,
        show_default=False,
    )

    template_keys = list(TEMPLATES.keys())
    if 1 <= template_choice <= len(template_keys):
        selected_template: str = template_keys[template_choice - 1]
    else:
        click.secho("❌ Invalid choice. Please try again.", fg="red")
        return _choose_template()

    # Prompt the user to choose between Python or JS/TS version
    click.secho(
        f"\nYou selected: {selected_template} - {TEMPLATES[selected_template]['description']}",
        fg="green",
    )
    version_choice: int = click.prompt(
        "Choose language (1 for Python 🐍, 2 for JS/TS 🌐)", type=int
    )

    if version_choice == 1:
        return TEMPLATES[selected_template]["python"]
    elif version_choice == 2:
        return TEMPLATES[selected_template]["js"]
    else:
        click.secho("❌ Invalid choice. Please try again.", fg="red")
        return _choose_template()


def _download_repo_with_requests(repo_url: str, path: str) -> None:
    """Download a ZIP archive from the given URL and extracts it to the specified path.

    Args:
        repo_url: The URL of the repository to download.
        path: The path where the repository should be extracted.
    """
    click.secho("📥 Attempting to download repository as a ZIP archive...", fg="yellow")
    click.secho(f"URL: {repo_url}", fg="yellow")
    try:
        with request.urlopen(repo_url) as response:
            if response.status == 200:
                with ZipFile(BytesIO(response.read())) as zip_file:
                    zip_file.extractall(path)
                    # Move extracted contents to path
                    for item in os.listdir(path):
                        if item.endswith("-main"):
                            extracted_dir = os.path.join(path, item)
                            for filename in os.listdir(extracted_dir):
                                shutil.move(os.path.join(extracted_dir, filename), path)
                            shutil.rmtree(extracted_dir)
                click.secho(
                    f"✅ Downloaded and extracted repository to {path}", fg="green"
                )
    except error.HTTPError as e:
        click.secho(
            f"❌ Error: Failed to download repository.\nDetails: {e}\n",
            fg="red",
            bold=True,
            err=True,
        )
        sys.exit(1)


def _get_template_url(template_name: str) -> str | None:
    """
    Retrieves the template URL based on the provided template name.

    Args:
        template_name: The name of the template.

    Returns:
        Optional[str]: The URL of the template if found, else None.
    """
    if template_name in TEMPLATES:
        click.secho(f"Template selected: {template_name}", fg="green")
        version_choice: int = click.prompt(
            "Choose version (1 for Python 🐍, 2 for JS/TS 🌐)", type=int
        )

        if version_choice == 1:
            return TEMPLATES[template_name]["python"]
        elif version_choice == 2:
            return TEMPLATES[template_name]["js"]
        else:
            click.secho("❌ Invalid choice. Please try again.", fg="red")
            return None
    else:
        click.secho(
            f"Template '{template_name}' not found. Please select from the available options.",
            fg="red",
        )
        return None


def create_new(path: str | None, template: str | None) -> None:
    """Create a new LangGraph project at the specified PATH using the chosen TEMPLATE.

    Args:
        path: The path where the new project will be created.
        template: The name of the template to use.
    """
    # Prompt for path if not provided
    if not path:
        path = click.prompt(
            "📂 Please specify the path to create the application", default="."
        )

    path = os.path.abspath(path)  # Ensure path is absolute

    # Check if path exists and is not empty
    if os.path.exists(path) and os.listdir(path):
        click.secho(
            "❌ The specified directory already exists and is not empty. "
            "Aborting to prevent overwriting files.",
            fg="red",
            bold=True,
        )
        sys.exit(1)

    # Get template URL either from command-line argument or
    # through interactive selection
    if template:
        if template not in TEMPLATE_ID_TO_CONFIG:
            # Format available options in a readable way with descriptions
            template_options = ""
            for id_ in TEMPLATE_IDS:
                name, lang, _ = TEMPLATE_ID_TO_CONFIG[id_]
                description = TEMPLATES[name]["description"]

                # Add each template option with color formatting
                template_options += (
                    click.style("- ", fg="yellow", bold=True)
                    + click.style(f"{id_}", fg="cyan")
                    + click.style(f": {description}", fg="white")
                    + "\n"
                )

            # Display error message with colors and formatting
            click.secho("❌ Error:", fg="red", bold=True, nl=False)
            click.secho(f" Template '{template}' not found.", fg="red")
            click.secho(
                "Please select from the available options:\n", fg="yellow", bold=True
            )
            click.secho(template_options, fg="cyan")
            sys.exit(1)
        _, _, template_url = TEMPLATE_ID_TO_CONFIG[template]
    else:
        template_url = _choose_template()

    # Download and extract the template
    _download_repo_with_requests(template_url, path)

    click.secho(f"🎉 New project created at {path}", fg="green", bold=True)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/util.py
```py
import click


def clean_empty_lines(input_str: str):
    return "\n".join(filter(None, input_str.splitlines()))


def warn_non_wolfi_distro(config_json: dict) -> None:
    """Show warning if image_distro is not set to 'wolfi'."""
    image_distro = config_json.get("image_distro", "debian")  # Default is debian
    if image_distro != "wolfi":
        click.secho(
            "⚠️  Security Recommendation: Consider switching to Wolfi Linux for enhanced security.",
            fg="yellow",
            bold=True,
        )
        click.secho(
            "   Wolfi is a security-oriented, minimal Linux distribution designed for containers.",
            fg="yellow",
        )
        click.secho(
            '   To switch, add \'"image_distro": "wolfi"\' to your langgraph.json config file.',
            fg="yellow",
        )
        click.secho("")  # Empty line for better readability

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/version.py
```py
"""Main entrypoint into package."""

from importlib import metadata

try:
    __version__ = metadata.version(__package__)
except metadata.PackageNotFoundError:
    # Case where package metadata is not available.
    __version__ = ""
del metadata  # optional, avoids polluting the results of dir(__package__)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/js-examples/.dockerignore
```text
node_modules
dist
```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/js-examples/.editorconfig
```text
root = true

[*]
end_of_line = lf
insert_final_newline = true

[*.{js,json,yml}]
charset = utf-8
indent_style = space
indent_size = 2

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/js-examples/.env.example
```example
# Copy this over:
# cp .env.example .env
# Then modify to suit your needs
```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/js-examples/.eslintrc.cjs
```cjs
module.exports = {
  extends: [
    "eslint:recommended",
    "prettier",
    "plugin:@typescript-eslint/recommended",
  ],
  parserOptions: {
    ecmaVersion: 12,
    parser: "@typescript-eslint/parser",
    project: "./tsconfig.json",
    sourceType: "module",
  },
  plugins: ["import", "@typescript-eslint", "no-instanceof"],
  ignorePatterns: [
    ".eslintrc.cjs",
    "scripts",
    "src/utils/lodash/*",
    "node_modules",
    "dist",
    "dist-cjs",
    "*.js",
    "*.cjs",
    "*.d.ts",
  ],
  rules: {
    "no-process-env": 2,
    "no-instanceof/no-instanceof": 2,
    "@typescript-eslint/explicit-module-boundary-types": 0,
    "@typescript-eslint/no-empty-function": 0,
    "@typescript-eslint/no-shadow": 0,
    "@typescript-eslint/no-empty-interface": 0,
    "@typescript-eslint/no-use-before-define": ["error", "nofunc"],
    "@typescript-eslint/no-unused-vars": ["warn", { args: "none" }],
    "@typescript-eslint/no-floating-promises": "error",
    "@typescript-eslint/no-misused-promises": "error",
    camelcase: 0,
    "class-methods-use-this": 0,
    "import/extensions": [2, "ignorePackages"],
    "import/no-extraneous-dependencies": [
      "error",
      { devDependencies: ["**/*.test.ts"] },
    ],
    "import/no-unresolved": 0,
    "import/prefer-default-export": 0,
    "keyword-spacing": "error",
    "max-classes-per-file": 0,
    "max-len": 0,
    "no-await-in-loop": 0,
    "no-bitwise": 0,
    "no-console": 0,
    "no-restricted-syntax": 0,
    "no-shadow": 0,
    "no-continue": 0,
    "no-underscore-dangle": 0,
    "no-use-before-define": 0,
    "no-useless-constructor": 0,
    "no-return-await": 0,
    "consistent-return": 0,
    "no-else-return": 0,
    "new-cap": ["error", { properties: false, capIsNew: false }],
  },
};

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/js-examples/.gitignore
```text
index.cjs
index.js
index.d.ts
node_modules
dist
.yarn/*
!.yarn/patches
!.yarn/plugins
!.yarn/releases
!.yarn/sdks
!.yarn/versions

.turbo
**/.turbo
**/.eslintcache

.env
.ipynb_checkpoints


```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/js-examples/LICENSE
```text
MIT License

Copyright (c) 2024 LangChain

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/js-examples/README.md
```md
# New LangGraph.js Project

[![CI](https://github.com/langchain-ai/new-langgraphjs-project/actions/workflows/unit-tests.yml/badge.svg)](https://github.com/langchain-ai/new-langgraphjs-project/actions/workflows/unit-tests.yml)
[![Integration Tests](https://github.com/langchain-ai/new-langgraphjs-project/actions/workflows/integration-tests.yml/badge.svg)](https://github.com/langchain-ai/new-langgraphjs-project/actions/workflows/integration-tests.yml)
[![Open in - LangGraph Studio](https://img.shields.io/badge/Open_in-LangGraph_Studio-00324d.svg?logo=data:image/svg%2bxml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSI4NS4zMzMiIGhlaWdodD0iODUuMzMzIiB2ZXJzaW9uPSIxLjAiIHZpZXdCb3g9IjAgMCA2NCA2NCI+PHBhdGggZD0iTTEzIDcuOGMtNi4zIDMuMS03LjEgNi4zLTYuOCAyNS43LjQgMjQuNi4zIDI0LjUgMjUuOSAyNC41QzU3LjUgNTggNTggNTcuNSA1OCAzMi4zIDU4IDcuMyA1Ni43IDYgMzIgNmMtMTIuOCAwLTE2LjEuMy0xOSAxLjhtMzcuNiAxNi42YzIuOCAyLjggMy40IDQuMiAzLjQgNy42cy0uNiA0LjgtMy40IDcuNkw0Ny4yIDQzSDE2LjhsLTMuNC0zLjRjLTQuOC00LjgtNC44LTEwLjQgMC0xNS4ybDMuNC0zLjRoMzAuNHoiLz48cGF0aCBkPSJNMTguOSAyNS42Yy0xLjEgMS4zLTEgMS43LjQgMi41LjkuNiAxLjcgMS44IDEuNyAyLjcgMCAxIC43IDIuOCAxLjYgNC4xIDEuNCAxLjkgMS40IDIuNS4zIDMuMi0xIC42LS42LjkgMS40LjkgMS41IDAgMi43LS41IDIuNy0xIDAtLjYgMS4xLS44IDIuNi0uNGwyLjYuNy0xLjgtMi45Yy01LjktOS4zLTkuNC0xMi4zLTExLjUtOS44TTM5IDI2YzAgMS4xLS45IDIuNS0yIDMuMi0yLjQgMS41LTIuNiAzLjQtLjUgNC4yLjguMyAyIDEuNyAyLjUgMy4xLjYgMS41IDEuNCAyLjMgMiAyIDEuNS0uOSAxLjItMy41LS40LTMuNS0yLjEgMC0yLjgtMi44LS44LTMuMyAxLjYtLjQgMS42LS41IDAtLjYtMS4xLS4xLTEuNS0uNi0xLjItMS42LjctMS43IDMuMy0yLjEgMy41LS41LjEuNS4yIDEuNi4zIDIuMiAwIC43LjkgMS40IDEuOSAxLjYgMi4xLjQgMi4zLTIuMy4yLTMuMi0uOC0uMy0yLTEuNy0yLjUtMy4xLTEuMS0zLTMtMy4zLTMtLjUiLz48L3N2Zz4=)](https://langgraph-studio.vercel.app/templates/open?githubUrl=https://github.com/langchain-ai/new-langgraphjs-project)

This template demonstrates a simple chatbot implemented using [LangGraph.js](https://github.com/langchain-ai/langgraphjs), designed for [LangGraph Studio](https://github.com/langchain-ai/langgraph-studio). The chatbot maintains persistent chat memory, allowing for coherent conversations across multiple interactions.

![Graph view in LangGraph studio UI](./static/studio.png)

The core logic, defined in `src/agent/graph.ts`, showcases a straightforward chatbot that responds to user queries while maintaining context from previous messages.

## What it does

The simple chatbot:

1. Takes a user **message** as input
2. Maintains a history of the conversation
3. Returns a placeholder response, updating the conversation history

This template provides a foundation that can be easily customized and extended to create more complex conversational agents.

## Getting Started

Assuming you have already [installed LangGraph Studio](https://github.com/langchain-ai/langgraph-studio?tab=readme-ov-file#download), to set up:

1. Create a `.env` file. This template does not require any environment variables by default, but you will likely want to add some when customizing.

```bash
cp .env.example .env
```

<!--
Setup instruction auto-generated by `langgraph template lock`. DO NOT EDIT MANUALLY.
-->

<!--
End setup instructions
-->

2. Open the folder in LangGraph Studio!
3. Customize the code as needed.

## How to customize

1. **Add an LLM call**: You can select and install a chat model wrapper from [the LangChain.js ecosystem](https://js.langchain.com/docs/integrations/chat/), or use LangGraph.js without LangChain.js.
2. **Extend the graph**: The core logic of the chatbot is defined in [graph.ts](./src/agent/graph.ts). You can modify this file to add new nodes, edges, or change the flow of the conversation.

You can also extend this template by:

- Adding [custom tools or functions](https://js.langchain.com/docs/how_to/tool_calling) to enhance the chatbot's capabilities.
- Implementing additional logic for handling specific types of user queries or tasks.
- Add retrieval-augmented generation (RAG) capabilities by integrating [external APIs or databases](https://langchain-ai.github.io/langgraphjs/tutorials/rag/langgraph_agentic_rag/) to provide more customized responses.

## Development

While iterating on your graph, you can edit past state and rerun your app from previous states to debug specific nodes. Local changes will be automatically applied via hot reload. Try experimenting with:

- Modifying the system prompt to give your chatbot a unique personality.
- Adding new nodes to the graph for more complex conversation flows.
- Implementing conditional logic to handle different types of user inputs.

Follow-up requests will be appended to the same thread. You can create an entirely new thread, clearing previous history, using the `+` button in the top right.

For more advanced features and examples, refer to the [LangGraph.js documentation](https://github.com/langchain-ai/langgraphjs). These resources can help you adapt this template for your specific use case and build more sophisticated conversational agents.

LangGraph Studio also integrates with [LangSmith](https://smith.langchain.com/) for more in-depth tracing and collaboration with teammates, allowing you to analyze and optimize your chatbot's performance.

<!--
Configuration auto-generated by `langgraph template lock`. DO NOT EDIT MANUALLY.
{
  "config_schemas": {
    "agent": {
      "type": "object",
      "properties": {}
    }
  }
}
-->

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/js-examples/jest.config.js
```js
export default {
  preset: "ts-jest/presets/default-esm",
  moduleNameMapper: {
    "^(\\.{1,2}/.*)\\.js$": "$1",
  },
  transform: {
    "^.+\\.tsx?$": [
      "ts-jest",
      {
        useESM: true,
      },
    ],
  },
  extensionsToTreatAsEsm: [".ts"],
  setupFiles: ["dotenv/config"],
  passWithNoTests: true,
  testTimeout: 20_000,
};

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/js-examples/langgraph.json
```json
{
  "$schema": "https://langgra.ph/schema.json",
  "node_version": "20",
  "graphs": {
    "agent": "./src/agent/graph.ts:graph"
  },
  "env": ".env",
  "dependencies": ["."]
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/js-examples/package.json
```json
{
  "name": "example-graph",
  "version": "0.0.1",
  "description": "A starter template for creating a LangGraph workflow.",
  "packageManager": "yarn@1.22.22",
  "main": "my_app/graph.ts",
  "author": "Your Name",
  "license": "MIT",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "tsc",
    "clean": "rm -rf dist",
    "test": "node --experimental-vm-modules node_modules/jest/bin/jest.js --testPathPattern=\\.test\\.ts$ --testPathIgnorePatterns=\\.int\\.test\\.ts$",
    "test:int": "node --experimental-vm-modules node_modules/jest/bin/jest.js --testPathPattern=\\.int\\.test\\.ts$",
    "format": "prettier --write .",
    "lint": "eslint src",
    "format:check": "prettier --check .",
    "lint:langgraph-json": "node scripts/checkLanggraphPaths.js",
    "lint:all": "yarn lint & yarn lint:langgraph-json & yarn format:check",
    "test:all": "yarn test && yarn test:int && yarn lint:langgraph"
  },
  "dependencies": {
    "@langchain/core": "^0.3.2",
    "@langchain/langgraph": "^0.2.5"
  },
  "devDependencies": {
    "@eslint/eslintrc": "^3.1.0",
    "@eslint/js": "^9.9.1",
    "@tsconfig/recommended": "^1.0.7",
    "@types/jest": "^29.5.0",
    "@typescript-eslint/eslint-plugin": "^5.59.8",
    "@typescript-eslint/parser": "^5.59.8",
    "dotenv": "^16.4.5",
    "eslint": "^8.41.0",
    "eslint-config-prettier": "^8.8.0",
    "eslint-plugin-import": "^2.27.5",
    "eslint-plugin-no-instanceof": "^1.0.1",
    "eslint-plugin-prettier": "^4.2.1",
    "jest": "^29.7.0",
    "prettier": "^3.3.3",
    "ts-jest": "^29.1.0",
    "typescript": "^5.3.3"
  }
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/js-examples/tsconfig.json
```json
{
  "extends": "@tsconfig/recommended",
  "compilerOptions": {
    "target": "ES2021",
    "lib": ["ES2021", "ES2022.Object", "DOM"],
    "module": "NodeNext",
    "moduleResolution": "nodenext",
    "esModuleInterop": true,
    "noImplicitReturns": true,
    "declaration": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "useDefineForClassFields": true,
    "strictPropertyInitialization": false,
    "allowJs": true,
    "strict": true,
    "strictFunctionTypes": false,
    "outDir": "dist",
    "types": ["jest", "node"],
    "resolveJsonModule": true
  },
  "include": ["**/*.ts", "**/*.js"],
  "exclude": ["node_modules", "dist"]
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/js-examples/yarn.lock
```lock
# THIS IS AN AUTOGENERATED FILE. DO NOT EDIT THIS FILE DIRECTLY.
# yarn lockfile v1


"@ampproject/remapping@^2.2.0":
  version "2.3.0"
  resolved "https://registry.yarnpkg.com/@ampproject/remapping/-/remapping-2.3.0.tgz#ed441b6fa600072520ce18b43d2c8cc8caecc7f4"
  integrity sha512-30iZtAPgz+LTIYoeivqYo853f02jBYSd5uGnGpkFV0M3xOt9aN73erkgYAmZU43x4VfqcnLxW9Kpg3R5LC4YYw==
  dependencies:
    "@jridgewell/gen-mapping" "^0.3.5"
    "@jridgewell/trace-mapping" "^0.3.24"

"@babel/code-frame@^7.0.0", "@babel/code-frame@^7.12.13", "@babel/code-frame@^7.24.7":
  version "7.24.7"
  resolved "https://registry.yarnpkg.com/@babel/code-frame/-/code-frame-7.24.7.tgz#882fd9e09e8ee324e496bd040401c6f046ef4465"
  integrity sha512-BcYH1CVJBO9tvyIZ2jVeXgSIMvGZ2FDRvDdOIVQyuklNKSsx+eppDEBq/g47Ayw+RqNFE+URvOShmf+f/qwAlA==
  dependencies:
    "@babel/highlight" "^7.24.7"
    picocolors "^1.0.0"

"@babel/compat-data@^7.25.2":
  version "7.25.2"
  resolved "https://registry.yarnpkg.com/@babel/compat-data/-/compat-data-7.25.2.tgz#e41928bd33475305c586f6acbbb7e3ade7a6f7f5"
  integrity sha512-bYcppcpKBvX4znYaPEeFau03bp89ShqNMLs+rmdptMw+heSZh9+z84d2YG+K7cYLbWwzdjtDoW/uqZmPjulClQ==

"@babel/core@^7.11.6", "@babel/core@^7.12.3", "@babel/core@^7.23.9":
  version "7.25.2"
  resolved "https://registry.yarnpkg.com/@babel/core/-/core-7.25.2.tgz#ed8eec275118d7613e77a352894cd12ded8eba77"
  integrity sha512-BBt3opiCOxUr9euZ5/ro/Xv8/V7yJ5bjYMqG/C1YAo8MIKAnumZalCN+msbci3Pigy4lIQfPUpfMM27HMGaYEA==
  dependencies:
    "@ampproject/remapping" "^2.2.0"
    "@babel/code-frame" "^7.24.7"
    "@babel/generator" "^7.25.0"
    "@babel/helper-compilation-targets" "^7.25.2"
    "@babel/helper-module-transforms" "^7.25.2"
    "@babel/helpers" "^7.25.0"
    "@babel/parser" "^7.25.0"
    "@babel/template" "^7.25.0"
    "@babel/traverse" "^7.25.2"
    "@babel/types" "^7.25.2"
    convert-source-map "^2.0.0"
    debug "^4.1.0"
    gensync "^1.0.0-beta.2"
    json5 "^2.2.3"
    semver "^6.3.1"

"@babel/generator@^7.25.0", "@babel/generator@^7.7.2":
  version "7.25.0"
  resolved "https://registry.yarnpkg.com/@babel/generator/-/generator-7.25.0.tgz#f858ddfa984350bc3d3b7f125073c9af6988f18e"
  integrity sha512-3LEEcj3PVW8pW2R1SR1M89g/qrYk/m/mB/tLqn7dn4sbBUQyTqnlod+II2U4dqiGtUmkcnAmkMDralTFZttRiw==
  dependencies:
    "@babel/types" "^7.25.0"
    "@jridgewell/gen-mapping" "^0.3.5"
    "@jridgewell/trace-mapping" "^0.3.25"
    jsesc "^2.5.1"

"@babel/helper-compilation-targets@^7.25.2":
  version "7.25.2"
  resolved "https://registry.yarnpkg.com/@babel/helper-compilation-targets/-/helper-compilation-targets-7.25.2.tgz#e1d9410a90974a3a5a66e84ff55ef62e3c02d06c"
  integrity sha512-U2U5LsSaZ7TAt3cfaymQ8WHh0pxvdHoEk6HVpaexxixjyEquMh0L0YNJNM6CTGKMXV1iksi0iZkGw4AcFkPaaw==
  dependencies:
    "@babel/compat-data" "^7.25.2"
    "@babel/helper-validator-option" "^7.24.8"
    browserslist "^4.23.1"
    lru-cache "^5.1.1"
    semver "^6.3.1"

"@babel/helper-module-imports@^7.24.7":
  version "7.24.7"
  resolved "https://registry.yarnpkg.com/@babel/helper-module-imports/-/helper-module-imports-7.24.7.tgz#f2f980392de5b84c3328fc71d38bd81bbb83042b"
  integrity sha512-8AyH3C+74cgCVVXow/myrynrAGv+nTVg5vKu2nZph9x7RcRwzmh0VFallJuFTZ9mx6u4eSdXZfcOzSqTUm0HCA==
  dependencies:
    "@babel/traverse" "^7.24.7"
    "@babel/types" "^7.24.7"

"@babel/helper-module-transforms@^7.25.2":
  version "7.25.2"
  resolved "https://registry.yarnpkg.com/@babel/helper-module-transforms/-/helper-module-transforms-7.25.2.tgz#ee713c29768100f2776edf04d4eb23b8d27a66e6"
  integrity sha512-BjyRAbix6j/wv83ftcVJmBt72QtHI56C7JXZoG2xATiLpmoC7dpd8WnkikExHDVPpi/3qCmO6WY1EaXOluiecQ==
  dependencies:
    "@babel/helper-module-imports" "^7.24.7"
    "@babel/helper-simple-access" "^7.24.7"
    "@babel/helper-validator-identifier" "^7.24.7"
    "@babel/traverse" "^7.25.2"

"@babel/helper-plugin-utils@^7.0.0", "@babel/helper-plugin-utils@^7.10.4", "@babel/helper-plugin-utils@^7.12.13", "@babel/helper-plugin-utils@^7.14.5", "@babel/helper-plugin-utils@^7.24.7", "@babel/helper-plugin-utils@^7.8.0":
  version "7.24.8"
  resolved "https://registry.yarnpkg.com/@babel/helper-plugin-utils/-/helper-plugin-utils-7.24.8.tgz#94ee67e8ec0e5d44ea7baeb51e571bd26af07878"
  integrity sha512-FFWx5142D8h2Mgr/iPVGH5G7w6jDn4jUSpZTyDnQO0Yn7Ks2Kuz6Pci8H6MPCoUJegd/UZQ3tAvfLCxQSnWWwg==

"@babel/helper-simple-access@^7.24.7":
  version "7.24.7"
  resolved "https://registry.yarnpkg.com/@babel/helper-simple-access/-/helper-simple-access-7.24.7.tgz#bcade8da3aec8ed16b9c4953b74e506b51b5edb3"
  integrity sha512-zBAIvbCMh5Ts+b86r/CjU+4XGYIs+R1j951gxI3KmmxBMhCg4oQMsv6ZXQ64XOm/cvzfU1FmoCyt6+owc5QMYg==
  dependencies:
    "@babel/traverse" "^7.24.7"
    "@babel/types" "^7.24.7"

"@babel/helper-string-parser@^7.24.8":
  version "7.24.8"
  resolved "https://registry.yarnpkg.com/@babel/helper-string-parser/-/helper-string-parser-7.24.8.tgz#5b3329c9a58803d5df425e5785865881a81ca48d"
  integrity sha512-pO9KhhRcuUyGnJWwyEgnRJTSIZHiT+vMD0kPeD+so0l7mxkMT19g3pjY9GTnHySck/hDzq+dtW/4VgnMkippsQ==

"@babel/helper-validator-identifier@^7.24.7":
  version "7.24.7"
  resolved "https://registry.yarnpkg.com/@babel/helper-validator-identifier/-/helper-validator-identifier-7.24.7.tgz#75b889cfaf9e35c2aaf42cf0d72c8e91719251db"
  integrity sha512-rR+PBcQ1SMQDDyF6X0wxtG8QyLCgUB0eRAGguqRLfkCA87l7yAP7ehq8SNj96OOGTO8OBV70KhuFYcIkHXOg0w==

"@babel/helper-validator-option@^7.24.8":
  version "7.24.8"
  resolved "https://registry.yarnpkg.com/@babel/helper-validator-option/-/helper-validator-option-7.24.8.tgz#3725cdeea8b480e86d34df15304806a06975e33d"
  integrity sha512-xb8t9tD1MHLungh/AIoWYN+gVHaB9kwlu8gffXGSt3FFEIT7RjS+xWbc2vUD1UTZdIpKj/ab3rdqJ7ufngyi2Q==

"@babel/helpers@^7.25.0":
  version "7.25.0"
  resolved "https://registry.yarnpkg.com/@babel/helpers/-/helpers-7.25.0.tgz#e69beb7841cb93a6505531ede34f34e6a073650a"
  integrity sha512-MjgLZ42aCm0oGjJj8CtSM3DB8NOOf8h2l7DCTePJs29u+v7yO/RBX9nShlKMgFnRks/Q4tBAe7Hxnov9VkGwLw==
  dependencies:
    "@babel/template" "^7.25.0"
    "@babel/types" "^7.25.0"

"@babel/highlight@^7.24.7":
  version "7.24.7"
  resolved "https://registry.yarnpkg.com/@babel/highlight/-/highlight-7.24.7.tgz#a05ab1df134b286558aae0ed41e6c5f731bf409d"
  integrity sha512-EStJpq4OuY8xYfhGVXngigBJRWxftKX9ksiGDnmlY3o7B/V7KIAc9X4oiK87uPJSc/vs5L869bem5fhZa8caZw==
  dependencies:
    "@babel/helper-validator-identifier" "^7.24.7"
    chalk "^2.4.2"
    js-tokens "^4.0.0"
    picocolors "^1.0.0"

"@babel/parser@^7.1.0", "@babel/parser@^7.14.7", "@babel/parser@^7.20.7", "@babel/parser@^7.23.9", "@babel/parser@^7.25.0", "@babel/parser@^7.25.3":
  version "7.25.3"
  resolved "https://registry.yarnpkg.com/@babel/parser/-/parser-7.25.3.tgz#91fb126768d944966263f0657ab222a642b82065"
  integrity sha512-iLTJKDbJ4hMvFPgQwwsVoxtHyWpKKPBrxkANrSYewDPaPpT5py5yeVkgPIJ7XYXhndxJpaA3PyALSXQ7u8e/Dw==
  dependencies:
    "@babel/types" "^7.25.2"

"@babel/plugin-syntax-async-generators@^7.8.4":
  version "7.8.4"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-async-generators/-/plugin-syntax-async-generators-7.8.4.tgz#a983fb1aeb2ec3f6ed042a210f640e90e786fe0d"
  integrity sha512-tycmZxkGfZaxhMRbXlPXuVFpdWlXpir2W4AMhSJgRKzk/eDlIXOhb2LHWoLpDF7TEHylV5zNhykX6KAgHJmTNw==
  dependencies:
    "@babel/helper-plugin-utils" "^7.8.0"

"@babel/plugin-syntax-bigint@^7.8.3":
  version "7.8.3"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-bigint/-/plugin-syntax-bigint-7.8.3.tgz#4c9a6f669f5d0cdf1b90a1671e9a146be5300cea"
  integrity sha512-wnTnFlG+YxQm3vDxpGE57Pj0srRU4sHE/mDkt1qv2YJJSeUAec2ma4WLUnUPeKjyrfntVwe/N6dCXpU+zL3Npg==
  dependencies:
    "@babel/helper-plugin-utils" "^7.8.0"

"@babel/plugin-syntax-class-properties@^7.12.13":
  version "7.12.13"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-class-properties/-/plugin-syntax-class-properties-7.12.13.tgz#b5c987274c4a3a82b89714796931a6b53544ae10"
  integrity sha512-fm4idjKla0YahUNgFNLCB0qySdsoPiZP3iQE3rky0mBUtMZ23yDJ9SJdg6dXTSDnulOVqiF3Hgr9nbXvXTQZYA==
  dependencies:
    "@babel/helper-plugin-utils" "^7.12.13"

"@babel/plugin-syntax-class-static-block@^7.14.5":
  version "7.14.5"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-class-static-block/-/plugin-syntax-class-static-block-7.14.5.tgz#195df89b146b4b78b3bf897fd7a257c84659d406"
  integrity sha512-b+YyPmr6ldyNnM6sqYeMWE+bgJcJpO6yS4QD7ymxgH34GBPNDM/THBh8iunyvKIZztiwLH4CJZ0RxTk9emgpjw==
  dependencies:
    "@babel/helper-plugin-utils" "^7.14.5"

"@babel/plugin-syntax-import-attributes@^7.24.7":
  version "7.24.7"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-import-attributes/-/plugin-syntax-import-attributes-7.24.7.tgz#b4f9ea95a79e6912480c4b626739f86a076624ca"
  integrity sha512-hbX+lKKeUMGihnK8nvKqmXBInriT3GVjzXKFriV3YC6APGxMbP8RZNFwy91+hocLXq90Mta+HshoB31802bb8A==
  dependencies:
    "@babel/helper-plugin-utils" "^7.24.7"

"@babel/plugin-syntax-import-meta@^7.10.4":
  version "7.10.4"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-import-meta/-/plugin-syntax-import-meta-7.10.4.tgz#ee601348c370fa334d2207be158777496521fd51"
  integrity sha512-Yqfm+XDx0+Prh3VSeEQCPU81yC+JWZ2pDPFSS4ZdpfZhp4MkFMaDC1UqseovEKwSUpnIL7+vK+Clp7bfh0iD7g==
  dependencies:
    "@babel/helper-plugin-utils" "^7.10.4"

"@babel/plugin-syntax-json-strings@^7.8.3":
  version "7.8.3"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-json-strings/-/plugin-syntax-json-strings-7.8.3.tgz#01ca21b668cd8218c9e640cb6dd88c5412b2c96a"
  integrity sha512-lY6kdGpWHvjoe2vk4WrAapEuBR69EMxZl+RoGRhrFGNYVK8mOPAW8VfbT/ZgrFbXlDNiiaxQnAtgVCZ6jv30EA==
  dependencies:
    "@babel/helper-plugin-utils" "^7.8.0"

"@babel/plugin-syntax-jsx@^7.7.2":
  version "7.24.7"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-jsx/-/plugin-syntax-jsx-7.24.7.tgz#39a1fa4a7e3d3d7f34e2acc6be585b718d30e02d"
  integrity sha512-6ddciUPe/mpMnOKv/U+RSd2vvVy+Yw/JfBB0ZHYjEZt9NLHmCUylNYlsbqCCS1Bffjlb0fCwC9Vqz+sBz6PsiQ==
  dependencies:
    "@babel/helper-plugin-utils" "^7.24.7"

"@babel/plugin-syntax-logical-assignment-operators@^7.10.4":
  version "7.10.4"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-logical-assignment-operators/-/plugin-syntax-logical-assignment-operators-7.10.4.tgz#ca91ef46303530448b906652bac2e9fe9941f699"
  integrity sha512-d8waShlpFDinQ5MtvGU9xDAOzKH47+FFoney2baFIoMr952hKOLp1HR7VszoZvOsV/4+RRszNY7D17ba0te0ig==
  dependencies:
    "@babel/helper-plugin-utils" "^7.10.4"

"@babel/plugin-syntax-nullish-coalescing-operator@^7.8.3":
  version "7.8.3"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-nullish-coalescing-operator/-/plugin-syntax-nullish-coalescing-operator-7.8.3.tgz#167ed70368886081f74b5c36c65a88c03b66d1a9"
  integrity sha512-aSff4zPII1u2QD7y+F8oDsz19ew4IGEJg9SVW+bqwpwtfFleiQDMdzA/R+UlWDzfnHFCxxleFT0PMIrR36XLNQ==
  dependencies:
    "@babel/helper-plugin-utils" "^7.8.0"

"@babel/plugin-syntax-numeric-separator@^7.10.4":
  version "7.10.4"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-numeric-separator/-/plugin-syntax-numeric-separator-7.10.4.tgz#b9b070b3e33570cd9fd07ba7fa91c0dd37b9af97"
  integrity sha512-9H6YdfkcK/uOnY/K7/aA2xpzaAgkQn37yzWUMRK7OaPOqOpGS1+n0H5hxT9AUw9EsSjPW8SVyMJwYRtWs3X3ug==
  dependencies:
    "@babel/helper-plugin-utils" "^7.10.4"

"@babel/plugin-syntax-object-rest-spread@^7.8.3":
  version "7.8.3"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-object-rest-spread/-/plugin-syntax-object-rest-spread-7.8.3.tgz#60e225edcbd98a640332a2e72dd3e66f1af55871"
  integrity sha512-XoqMijGZb9y3y2XskN+P1wUGiVwWZ5JmoDRwx5+3GmEplNyVM2s2Dg8ILFQm8rWM48orGy5YpI5Bl8U1y7ydlA==
  dependencies:
    "@babel/helper-plugin-utils" "^7.8.0"

"@babel/plugin-syntax-optional-catch-binding@^7.8.3":
  version "7.8.3"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-optional-catch-binding/-/plugin-syntax-optional-catch-binding-7.8.3.tgz#6111a265bcfb020eb9efd0fdfd7d26402b9ed6c1"
  integrity sha512-6VPD0Pc1lpTqw0aKoeRTMiB+kWhAoT24PA+ksWSBrFtl5SIRVpZlwN3NNPQjehA2E/91FV3RjLWoVTglWcSV3Q==
  dependencies:
    "@babel/helper-plugin-utils" "^7.8.0"

"@babel/plugin-syntax-optional-chaining@^7.8.3":
  version "7.8.3"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-optional-chaining/-/plugin-syntax-optional-chaining-7.8.3.tgz#4f69c2ab95167e0180cd5336613f8c5788f7d48a"
  integrity sha512-KoK9ErH1MBlCPxV0VANkXW2/dw4vlbGDrFgz8bmUsBGYkFRcbRwMh6cIJubdPrkxRwuGdtCk0v/wPTKbQgBjkg==
  dependencies:
    "@babel/helper-plugin-utils" "^7.8.0"

"@babel/plugin-syntax-private-property-in-object@^7.14.5":
  version "7.14.5"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-private-property-in-object/-/plugin-syntax-private-property-in-object-7.14.5.tgz#0dc6671ec0ea22b6e94a1114f857970cd39de1ad"
  integrity sha512-0wVnp9dxJ72ZUJDV27ZfbSj6iHLoytYZmh3rFcxNnvsJF3ktkzLDZPy/mA17HGsaQT3/DQsWYX1f1QGWkCoVUg==
  dependencies:
    "@babel/helper-plugin-utils" "^7.14.5"

"@babel/plugin-syntax-top-level-await@^7.14.5":
  version "7.14.5"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-top-level-await/-/plugin-syntax-top-level-await-7.14.5.tgz#c1cfdadc35a646240001f06138247b741c34d94c"
  integrity sha512-hx++upLv5U1rgYfwe1xBQUhRmU41NEvpUvrp8jkrSCdvGSnM5/qdRMtylJ6PG5OFkBaHkbTAKTnd3/YyESRHFw==
  dependencies:
    "@babel/helper-plugin-utils" "^7.14.5"

"@babel/plugin-syntax-typescript@^7.7.2":
  version "7.24.7"
  resolved "https://registry.yarnpkg.com/@babel/plugin-syntax-typescript/-/plugin-syntax-typescript-7.24.7.tgz#58d458271b4d3b6bb27ee6ac9525acbb259bad1c"
  integrity sha512-c/+fVeJBB0FeKsFvwytYiUD+LBvhHjGSI0g446PRGdSVGZLRNArBUno2PETbAly3tpiNAQR5XaZ+JslxkotsbA==
  dependencies:
    "@babel/helper-plugin-utils" "^7.24.7"

"@babel/template@^7.25.0", "@babel/template@^7.3.3":
  version "7.25.0"
  resolved "https://registry.yarnpkg.com/@babel/template/-/template-7.25.0.tgz#e733dc3134b4fede528c15bc95e89cb98c52592a"
  integrity sha512-aOOgh1/5XzKvg1jvVz7AVrx2piJ2XBi227DHmbY6y+bM9H2FlN+IfecYu4Xl0cNiiVejlsCri89LUsbj8vJD9Q==
  dependencies:
    "@babel/code-frame" "^7.24.7"
    "@babel/parser" "^7.25.0"
    "@babel/types" "^7.25.0"

"@babel/traverse@^7.24.7", "@babel/traverse@^7.25.2":
  version "7.25.3"
  resolved "https://registry.yarnpkg.com/@babel/traverse/-/traverse-7.25.3.tgz#f1b901951c83eda2f3e29450ce92743783373490"
  integrity sha512-HefgyP1x754oGCsKmV5reSmtV7IXj/kpaE1XYY+D9G5PvKKoFfSbiS4M77MdjuwlZKDIKFCffq9rPU+H/s3ZdQ==
  dependencies:
    "@babel/code-frame" "^7.24.7"
    "@babel/generator" "^7.25.0"
    "@babel/parser" "^7.25.3"
    "@babel/template" "^7.25.0"
    "@babel/types" "^7.25.2"
    debug "^4.3.1"
    globals "^11.1.0"

"@babel/types@^7.0.0", "@babel/types@^7.20.7", "@babel/types@^7.24.7", "@babel/types@^7.25.0", "@babel/types@^7.25.2", "@babel/types@^7.3.3":
  version "7.25.2"
  resolved "https://registry.yarnpkg.com/@babel/types/-/types-7.25.2.tgz#55fb231f7dc958cd69ea141a4c2997e819646125"
  integrity sha512-YTnYtra7W9e6/oAZEHj0bJehPRUlLH9/fbpT5LfB0NhQXyALCRkRs3zH9v07IYhkgpqX6Z78FnuccZr/l4Fs4Q==
  dependencies:
    "@babel/helper-string-parser" "^7.24.8"
    "@babel/helper-validator-identifier" "^7.24.7"
    to-fast-properties "^2.0.0"

"@bcoe/v8-coverage@^0.2.3":
  version "0.2.3"
  resolved "https://registry.yarnpkg.com/@bcoe/v8-coverage/-/v8-coverage-0.2.3.tgz#75a2e8b51cb758a7553d6804a5932d7aace75c39"
  integrity sha512-0hYQ8SB4Db5zvZB4axdMHGwEaQjkZzFjQiN9LVYvIFB2nSUHW9tYpxWriPrWDASIxiaXax83REcLxuSdnGPZtw==

"@eslint-community/eslint-utils@^4.2.0":
  version "4.4.0"
  resolved "https://registry.yarnpkg.com/@eslint-community/eslint-utils/-/eslint-utils-4.4.0.tgz#a23514e8fb9af1269d5f7788aa556798d61c6b59"
  integrity sha512-1/sA4dwrzBAyeUoQ6oxahHKmrZvsnLCg4RfxW3ZFGGmQkSNQPFNLV9CUEFQP1x9EYXHTo5p6xdhZM1Ne9p/AfA==
  dependencies:
    eslint-visitor-keys "^3.3.0"

"@eslint-community/regexpp@^4.4.0", "@eslint-community/regexpp@^4.6.1":
  version "4.11.0"
  resolved "https://registry.yarnpkg.com/@eslint-community/regexpp/-/regexpp-4.11.0.tgz#b0ffd0312b4a3fd2d6f77237e7248a5ad3a680ae"
  integrity sha512-G/M/tIiMrTAxEWRfLfQJMmGNX28IxBg4PBz8XqQhqUHLFI6TL2htpIB1iQCj144V5ee/JaKyT9/WZ0MGZWfA7A==

"@eslint/eslintrc@^2.1.4":
  version "2.1.4"
  resolved "https://registry.yarnpkg.com/@eslint/eslintrc/-/eslintrc-2.1.4.tgz#388a269f0f25c1b6adc317b5a2c55714894c70ad"
  integrity sha512-269Z39MS6wVJtsoUl10L60WdkhJVdPG24Q4eZTH3nnF6lpvSShEK3wQjDX9JRWAUPvPh7COouPpU9IrqaZFvtQ==
  dependencies:
    ajv "^6.12.4"
    debug "^4.3.2"
    espree "^9.6.0"
    globals "^13.19.0"
    ignore "^5.2.0"
    import-fresh "^3.2.1"
    js-yaml "^4.1.0"
    minimatch "^3.1.2"
    strip-json-comments "^3.1.1"

"@eslint/eslintrc@^3.1.0":
  version "3.1.0"
  resolved "https://registry.yarnpkg.com/@eslint/eslintrc/-/eslintrc-3.1.0.tgz#dbd3482bfd91efa663cbe7aa1f506839868207b6"
  integrity sha512-4Bfj15dVJdoy3RfZmmo86RK1Fwzn6SstsvK9JS+BaVKqC6QQQQyXekNaC+g+LKNgkQ+2VhGAzm6hO40AhMR3zQ==
  dependencies:
    ajv "^6.12.4"
    debug "^4.3.2"
    espree "^10.0.1"
    globals "^14.0.0"
    ignore "^5.2.0"
    import-fresh "^3.2.1"
    js-yaml "^4.1.0"
    minimatch "^3.1.2"
    strip-json-comments "^3.1.1"

"@eslint/js@8.57.0":
  version "8.57.0"
  resolved "https://registry.yarnpkg.com/@eslint/js/-/js-8.57.0.tgz#a5417ae8427873f1dd08b70b3574b453e67b5f7f"
  integrity sha512-Ys+3g2TaW7gADOJzPt83SJtCDhMjndcDMFVQ/Tj9iA1BfJzFKD9mAUXT3OenpuPHbI6P/myECxRJrofUsDx/5g==

"@eslint/js@^9.9.1":
  version "9.9.1"
  resolved "https://registry.yarnpkg.com/@eslint/js/-/js-9.9.1.tgz#4a97e85e982099d6c7ee8410aacb55adaa576f06"
  integrity sha512-xIDQRsfg5hNBqHz04H1R3scSVwmI+KUbqjsQKHKQ1DAUSaUjYPReZZmS/5PNiKu1fUvzDd6H7DEDKACSEhu+TQ==

"@humanwhocodes/config-array@^0.11.14":
  version "0.11.14"
  resolved "https://registry.yarnpkg.com/@humanwhocodes/config-array/-/config-array-0.11.14.tgz#d78e481a039f7566ecc9660b4ea7fe6b1fec442b"
  integrity sha512-3T8LkOmg45BV5FICb15QQMsyUSWrQ8AygVfC7ZG32zOalnqrilm018ZVCw0eapXux8FtA33q8PSRSstjee3jSg==
  dependencies:
    "@humanwhocodes/object-schema" "^2.0.2"
    debug "^4.3.1"
    minimatch "^3.0.5"

"@humanwhocodes/module-importer@^1.0.1":
  version "1.0.1"
  resolved "https://registry.yarnpkg.com/@humanwhocodes/module-importer/-/module-importer-1.0.1.tgz#af5b2691a22b44be847b0ca81641c5fb6ad0172c"
  integrity sha512-bxveV4V8v5Yb4ncFTT3rPSgZBOpCkjfK0y4oVVVJwIuDVBRMDXrPyXRL988i5ap9m9bnyEEjWfm5WkBmtffLfA==

"@humanwhocodes/object-schema@^2.0.2":
  version "2.0.3"
  resolved "https://registry.yarnpkg.com/@humanwhocodes/object-schema/-/object-schema-2.0.3.tgz#4a2868d75d6d6963e423bcf90b7fd1be343409d3"
  integrity sha512-93zYdMES/c1D69yZiKDBj0V24vqNzB/koF26KPaagAfd3P/4gUlh3Dys5ogAK+Exi9QyzlD8x/08Zt7wIKcDcA==

"@istanbuljs/load-nyc-config@^1.0.0":
  version "1.1.0"
  resolved "https://registry.yarnpkg.com/@istanbuljs/load-nyc-config/-/load-nyc-config-1.1.0.tgz#fd3db1d59ecf7cf121e80650bb86712f9b55eced"
  integrity sha512-VjeHSlIzpv/NyD3N0YuHfXOPDIixcA1q2ZV98wsMqcYlPmv2n3Yb2lYP9XMElnaFVXg5A7YLTeLu6V84uQDjmQ==
  dependencies:
    camelcase "^5.3.1"
    find-up "^4.1.0"
    get-package-type "^0.1.0"
    js-yaml "^3.13.1"
    resolve-from "^5.0.0"

"@istanbuljs/schema@^0.1.2", "@istanbuljs/schema@^0.1.3":
  version "0.1.3"
  resolved "https://registry.yarnpkg.com/@istanbuljs/schema/-/schema-0.1.3.tgz#e45e384e4b8ec16bce2fd903af78450f6bf7ec98"
  integrity sha512-ZXRY4jNvVgSVQ8DL3LTcakaAtXwTVUxE81hslsyD2AtoXW/wVob10HkOJ1X/pAlcI7D+2YoZKg5do8G/w6RYgA==

"@jest/console@^29.7.0":
  version "29.7.0"
  resolved "https://registry.yarnpkg.com/@jest/console/-/console-29.7.0.tgz#cd4822dbdb84529265c5a2bdb529a3c9cc950ffc"
  integrity sha512-5Ni4CU7XHQi32IJ398EEP4RrB8eV09sXP2ROqD4bksHrnTree52PsxvX8tpL8LvTZ3pFzXyPbNQReSN41CAhOg==
  dependencies:
    "@jest/types" "^29.6.3"
    "@types/node" "*"
    chalk "^4.0.0"
    jest-message-util "^29.7.0"
    jest-util "^29.7.0"
    slash "^3.0.0"

"@jest/core@^29.7.0":
  version "29.7.0"
  resolved "https://registry.yarnpkg.com/@jest/core/-/core-29.7.0.tgz#b6cccc239f30ff36609658c5a5e2291757ce448f"
  integrity sha512-n7aeXWKMnGtDA48y8TLWJPJmLmmZ642Ceo78cYWEpiD7FzDgmNDV/GCVRorPABdXLJZ/9wzzgZAlHjXjxDHGsg==
  dependencies:
    "@jest/console" "^29.7.0"
    "@jest/reporters" "^29.7.0"
    "@jest/test-result" "^29.7.0"
    "@jest/transform" "^29.7.0"
    "@jest/types" "^29.6.3"
    "@types/node" "*"
    ansi-escapes "^4.2.1"
    chalk "^4.0.0"
    ci-info "^3.2.0"
    exit "^0.1.2"
    graceful-fs "^4.2.9"
    jest-changed-files "^29.7.0"
    jest-config "^29.7.0"
    jest-haste-map "^29.7.0"
    jest-message-util "^29.7.0"
    jest-regex-util "^29.6.3"
    jest-resolve "^29.7.0"
    jest-resolve-dependencies "^29.7.0"
    jest-runner "^29.7.0"
    jest-runtime "^29.7.0"
    jest-snapshot "^29.7.0"
    jest-util "^29.7.0"
    jest-validate "^29.7.0"
    jest-watcher "^29.7.0"
    micromatch "^4.0.4"
    pretty-format "^29.7.0"
    slash "^3.0.0"
    strip-ansi "^6.0.0"

"@jest/environment@^29.7.0":
  version "29.7.0"
  resolved "https://registry.yarnpkg.com/@jest/environment/-/environment-29.7.0.tgz#24d61f54ff1f786f3cd4073b4b94416383baf2a7"
  integrity sha512-aQIfHDq33ExsN4jP1NWGXhxgQ/wixs60gDiKO+XVMd8Mn0NWPWgc34ZQDTb2jKaUWQ7MuwoitXAsN2XVXNMpAw==
  dependencies:
    "@jest/fake-timers" "^29.7.0"
    "@jest/types" "^29.6.3"
    "@types/node" "*"
    jest-mock "^29.7.0"

"@jest/expect-utils@^29.7.0":
  version "29.7.0"
  resolved "https://registry.yarnpkg.com/@jest/expect-utils/-/expect-utils-29.7.0.tgz#023efe5d26a8a70f21677d0a1afc0f0a44e3a1c6"
  integrity sha512-GlsNBWiFQFCVi9QVSx7f5AgMeLxe9YCCs5PuP2O2LdjDAA8Jh9eX7lA1Jq/xdXw3Wb3hyvlFNfZIfcRetSzYcA==
  dependencies:
    jest-get-type "^29.6.3"

"@jest/expect@^29.7.0":
  version "29.7.0"
  resolved "https://registry.yarnpkg.com/@jest/expect/-/expect-29.7.0.tgz#76a3edb0cb753b70dfbfe23283510d3d45432bf2"
  integrity sha512-8uMeAMycttpva3P1lBHB8VciS9V0XAr3GymPpipdyQXbBcuhkLQOSe8E/p92RyAdToS6ZD1tFkX+CkhoECE0dQ==
  dependencies:
    expect "^29.7.0"
    jest-snapshot "^29.7.0"

"@jest/fake-timers@^29.7.0":
  version "29.7.0"
  resolved "https://registry.yarnpkg.com/@jest/fake-timers/-/fake-timers-29.7.0.tgz#fd91bf1fffb16d7d0d24a426ab1a47a49881a565"
  integrity sha512-q4DH1Ha4TTFPdxLsqDXK1d3+ioSL7yL5oCMJZgDYm6i+6CygW5E5xVr/D1HdsGxjt1ZWSfUAs9OxSB/BNelWrQ==
  dependencies:
    "@jest/types" "^29.6.3"
    "@sinonjs/fake-timers" "^10.0.2"
    "@types/node" "*"
    jest-message-util "^29.7.0"
    jest-mock "^29.7.0"
    jest-util "^29.7.0"

"@jest/globals@^29.7.0":
  version "29.7.0"
  resolved "https://registry.yarnpkg.com/@jest/globals/-/globals-29.7.0.tgz#8d9290f9ec47ff772607fa864ca1d5a2efae1d4d"
  integrity sha512-mpiz3dutLbkW2MNFubUGUEVLkTGiqW6yLVTA+JbP6fI6J5iL9Y0Nlg8k95pcF8ctKwCS7WVxteBs29hhfAotzQ==
  dependencies:
    "@jest/environment" "^29.7.0"
    "@jest/expect" "^29.7.0"
    "@jest/types" "^29.6.3"
    jest-mock "^29.7.0"

"@jest/reporters@^29.7.0":
  version "29.7.0"
  resolved "https://registry.yarnpkg.com/@jest/reporters/-/reporters-29.7.0.tgz#04b262ecb3b8faa83b0b3d321623972393e8f4c7"
  integrity sha512-DApq0KJbJOEzAFYjHADNNxAE3KbhxQB1y5Kplb5Waqw6zVbuWatSnMjE5gs8FUgEPmNsnZA3NCWl9NG0ia04Pg==
  dependencies:
    "@bcoe/v8-coverage" "^0.2.3"
    "@jest/console" "^29.7.0"
    "@jest/test-result" "^29.7.0"
    "@jest/transform" "^29.7.0"
    "@jest/types" "^29.6.3"
    "@jridgewell/trace-mapping" "^0.3.18"
    "@types/node" "*"
    chalk "^4.0.0"
    collect-v8-coverage "^1.0.0"
    exit "^0.1.2"
    glob "^7.1.3"
    graceful-fs "^4.2.9"
    istanbul-lib-coverage "^3.0.0"
    istanbul-lib-instrument "^6.0.0"
    istanbul-lib-report "^3.0.0"
    istanbul-lib-source-maps "^4.0.0"
    istanbul-reports "^3.1.3"
    jest-message-util "^29.7.0"
    jest-util "^29.7.0"
    jest-worker "^29.7.0"
    slash "^3.0.0"
    string-length "^4.0.1"
    strip-ansi "^6.0.0"
    v8-to-istanbul "^9.0.1"

"@jest/schemas@^29.6.3":
  version "29.6.3"
  resolved "https://registry.yarnpkg.com/@jest/schemas/-/schemas-29.6.3.tgz#430b5ce8a4e0044a7e3819663305a7b3091c8e03"
  integrity sha512-mo5j5X+jIZmJQveBKeS/clAueipV7KgiX1vMgCxam1RNYiqE1w62n0/tJJnHtjW8ZHcQco5gY85jA3mi0L+nSA==
  dependencies:
    "@sinclair/typebox" "^0.27.8"

"@jest/source-map@^29.6.3":
  version "29.6.3"
  resolved "https://registry.yarnpkg.com/@jest/source-map/-/source-map-29.6.3.tgz#d90ba772095cf37a34a5eb9413f1b562a08554c4"
  integrity sha512-MHjT95QuipcPrpLM+8JMSzFx6eHp5Bm+4XeFDJlwsvVBjmKNiIAvasGK2fxz2WbGRlnvqehFbh07MMa7n3YJnw==
  dependencies:
    "@jridgewell/trace-mapping" "^0.3.18"
    callsites "^3.0.0"
    graceful-fs "^4.2.9"

"@jest/test-result@^29.7.0":
  version "29.7.0"
  resolved "https://registry.yarnpkg.com/@jest/test-result/-/test-result-29.7.0.tgz#8db9a80aa1a097bb2262572686734baed9b1657c"
  integrity sha512-Fdx+tv6x1zlkJPcWXmMDAG2HBnaR9XPSd5aDWQVsfrZmLVT3lU1cwyxLgRmXR9yrq4NBoEm9BMsfgFzTQAbJYA==
  dependencies:
    "@jest/console" "^29.7.0"
    "@jest/types" "^29.6.3"
    "@types/istanbul-lib-coverage" "^2.0.0"
    collect-v8-coverage "^1.0.0"

"@jest/test-sequencer@^29.7.0":
  version "29.7.0"
  resolved "https://registry.yarnpkg.com/@jest/test-sequencer/-/test-sequencer-29.7.0.tgz#6cef977ce1d39834a3aea887a1726628a6f072ce"
  integrity sha512-GQwJ5WZVrKnOJuiYiAF52UNUJXgTZx1NHjFSEB0qEMmSZKAkdMoIzw/Cj6x6NF4AvV23AUqDpFzQkN/eYCYTxw==
  dependencies:
    "@jest/test-result" "^29.7.0"
    graceful-fs "^4.2.9"
    jest-haste-map "^29.7.0"
    slash "^3.0.0"

"@jest/transform@^29.7.0":
  version "29.7.0"
  resolved "https://registry.yarnpkg.com/@jest/transform/-/transform-29.7.0.tgz#df2dd9c346c7d7768b8a06639994640c642e284c"
  integrity sha512-ok/BTPFzFKVMwO5eOHRrvnBVHdRy9IrsrW1GpMaQ9MCnilNLXQKmAX8s1YXDFaai9xJpac2ySzV0YeRRECr2Vw==
  dependencies:
    "@babel/core" "^7.11.6"
    "@jest/types" "^29.6.3"
    "@jridgewell/trace-mapping" "^0.3.18"
    babel-plugin-istanbul "^6.1.1"
    chalk "^4.0.0"
    convert-source-map "^2.0.0"
    fast-json-stable-stringify "^2.1.0"
    graceful-fs "^4.2.9"
    jest-haste-map "^29.7.0"
    jest-regex-util "^29.6.3"
    jest-util "^29.7.0"
    micromatch "^4.0.4"
    pirates "^4.0.4"
    slash "^3.0.0"
    write-file-atomic "^4.0.2"

"@jest/types@^29.6.3":
  version "29.6.3"
  resolved "https://registry.yarnpkg.com/@jest/types/-/types-29.6.3.tgz#1131f8cf634e7e84c5e77bab12f052af585fba59"
  integrity sha512-u3UPsIilWKOM3F9CXtrG8LEJmNxwoCQC/XVj4IKYXvvpx7QIi/Kg1LI5uDmDpKlac62NUtX7eLjRh+jVZcLOzw==
  dependencies:
    "@jest/schemas" "^29.6.3"
    "@types/istanbul-lib-coverage" "^2.0.0"
    "@types/istanbul-reports" "^3.0.0"
    "@types/node" "*"
    "@types/yargs" "^17.0.8"
    chalk "^4.0.0"

"@jridgewell/gen-mapping@^0.3.5":
  version "0.3.5"
  resolved "https://registry.yarnpkg.com/@jridgewell/gen-mapping/-/gen-mapping-0.3.5.tgz#dcce6aff74bdf6dad1a95802b69b04a2fcb1fb36"
  integrity sha512-IzL8ZoEDIBRWEzlCcRhOaCupYyN5gdIK+Q6fbFdPDg6HqX6jpkItn7DFIpW9LQzXG6Df9sA7+OKnq0qlz/GaQg==
  dependencies:
    "@jridgewell/set-array" "^1.2.1"
    "@jridgewell/sourcemap-codec" "^1.4.10"
    "@jridgewell/trace-mapping" "^0.3.24"

"@jridgewell/resolve-uri@^3.1.0":
  version "3.1.2"
  resolved "https://registry.yarnpkg.com/@jridgewell/resolve-uri/-/resolve-uri-3.1.2.tgz#7a0ee601f60f99a20c7c7c5ff0c80388c1189bd6"
  integrity sha512-bRISgCIjP20/tbWSPWMEi54QVPRZExkuD9lJL+UIxUKtwVJA8wW1Trb1jMs1RFXo1CBTNZ/5hpC9QvmKWdopKw==

"@jridgewell/set-array@^1.2.1":
  version "1.2.1"
  resolved "https://registry.yarnpkg.com/@jridgewell/set-array/-/set-array-1.2.1.tgz#558fb6472ed16a4c850b889530e6b36438c49280"
  integrity sha512-R8gLRTZeyp03ymzP/6Lil/28tGeGEzhx1q2k703KGWRAI1VdvPIXdG70VJc2pAMw3NA6JKL5hhFu1sJX0Mnn/A==

"@jridgewell/sourcemap-codec@^1.4.10", "@jridgewell/sourcemap-codec@^1.4.14":
  version "1.5.0"
  resolved "https://registry.yarnpkg.com/@jridgewell/sourcemap-codec/-/sourcemap-codec-1.5.0.tgz#3188bcb273a414b0d215fd22a58540b989b9409a"
  integrity sha512-gv3ZRaISU3fjPAgNsriBRqGWQL6quFx04YMPW/zD8XMLsU32mhCCbfbO6KZFLjvYpCZ8zyDEgqsgf+PwPaM7GQ==

"@jridgewell/trace-mapping@^0.3.12", "@jridgewell/trace-mapping@^0.3.18", "@jridgewell/trace-mapping@^0.3.24", "@jridgewell/trace-mapping@^0.3.25":
  version "0.3.25"
  resolved "https://registry.yarnpkg.com/@jridgewell/trace-mapping/-/trace-mapping-0.3.25.tgz#15f190e98895f3fc23276ee14bc76b675c2e50f0"
  integrity sha512-vNk6aEwybGtawWmy/PzwnGDOjCkLWSD2wqvjGGAgOAwCGWySYXfYoxt00IJkTF+8Lb57DwOb3Aa0o9CApepiYQ==
  dependencies:
    "@jridgewell/resolve-uri" "^3.1.0"
    "@jridgewell/sourcemap-codec" "^1.4.14"

"@langchain/core@^0.3.2":
  version "0.3.2"
  resolved "https://registry.yarnpkg.com/@langchain/core/-/core-0.3.2.tgz#aff6d83149a40e0e735910f583aca0f1dd7d1bab"
  integrity sha512-FeoDOStP8l1YdxgykpXnVoEnl4lxGNSOdYzUJN/EdFtkc6cIjDDS5+xewajme0+egaUsO4tGLezKaFpoWxAyQA==
  dependencies:
    ansi-styles "^5.0.0"
    camelcase "6"
    decamelize "1.2.0"
    js-tiktoken "^1.0.12"
    langsmith "^0.1.56"
    mustache "^4.2.0"
    p-queue "^6.6.2"
    p-retry "4"
    uuid "^10.0.0"
    zod "^3.22.4"
    zod-to-json-schema "^3.22.3"

"@langchain/langgraph-checkpoint@~0.0.17":
  version "0.0.17"
  resolved "https://registry.yarnpkg.com/@langchain/langgraph-checkpoint/-/langgraph-checkpoint-0.0.17.tgz#d0a8824eb0769567da54262adebe65db4ee6d58f"
  integrity sha512-6b3CuVVYx+7x0uWLG+7YXz9j2iBa+tn2AXvkLxzEvaAsLE6Sij++8PPbS2BZzC+S/FPJdWsz6I5bsrqL0BYrCA==
  dependencies:
    uuid "^10.0.0"

"@langchain/langgraph-sdk@~0.0.32":
  version "0.0.70"
  resolved "https://registry.yarnpkg.com/@langchain/langgraph-sdk/-/langgraph-sdk-0.0.70.tgz#9589f984b47de5e4a669b6008cbf01427a3d41d0"
  integrity sha512-O8I12bfeMVz5fOrXnIcK4IdRf50IqyJTO458V56wAIHLNoi4H8/JHM+2M+Y4H2PtslXIGnvomWqlBd0eY5z/Og==
  dependencies:
    "@types/json-schema" "^7.0.15"
    p-queue "^6.6.2"
    p-retry "4"
    uuid "^9.0.0"

"@langchain/langgraph@^0.2.5":
  version "0.2.67"
  resolved "https://registry.yarnpkg.com/@langchain/langgraph/-/langgraph-0.2.67.tgz#f154e4e8534ca78b2b73a2cd3a901d82790b0d1b"
  integrity sha512-tu/ewNIvhIPzeW5GxGzqjmGHinnU/qbNAoLM9czdpci0PCbMysbEJ2pbJrZs7ZjaReWSnr/THkeLPQwqGOM9xw==
  dependencies:
    "@langchain/langgraph-checkpoint" "~0.0.17"
    "@langchain/langgraph-sdk" "~0.0.32"
    uuid "^10.0.0"
    zod "^3.23.8"

"@nodelib/fs.scandir@2.1.5":
  version "2.1.5"
  resolved "https://registry.yarnpkg.com/@nodelib/fs.scandir/-/fs.scandir-2.1.5.tgz#7619c2eb21b25483f6d167548b4cfd5a7488c3d5"
  integrity sha512-vq24Bq3ym5HEQm2NKCr3yXDwjc7vTsEThRDnkp2DK9p1uqLR+DHurm/NOTo0KG7HYHU7eppKZj3MyqYuMBf62g==
  dependencies:
    "@nodelib/fs.stat" "2.0.5"
    run-parallel "^1.1.9"

"@nodelib/fs.stat@2.0.5", "@nodelib/fs.stat@^2.0.2":
  version "2.0.5"
  resolved "https://registry.yarnpkg.com/@nodelib/fs.stat/-/fs.stat-2.0.5.tgz#5bd262af94e9d25bd1e71b05deed44876a222e8b"
  integrity sha512-RkhPPp2zrqDAQA/2jNhnztcPAlv64XdhIp7a7454A5ovI7Bukxgt7MX7udwAu3zg1DcpPU0rz3VV1SeaqvY4+A==

"@nodelib/fs.walk@^1.2.3", "@nodelib/fs.walk@^1.2.8":
  version "1.2.8"
  resolved "https://registry.yarnpkg.com/@nodelib/fs.walk/-/fs.walk-1.2.8.tgz#e95737e8bb6746ddedf69c556953494f196fe69a"
  integrity sha512-oGB+UxlgWcgQkgwo8GcEGwemoTFt3FIO9ababBmaGwXIoBKZ+GTy0pP185beGg7Llih/NSHSV2XAs1lnznocSg==
  dependencies:
    "@nodelib/fs.scandir" "2.1.5"
    fastq "^1.6.0"

"@sinclair/typebox@^0.27.8":
  version "0.27.8"
  resolved "https://registry.yarnpkg.com/@sinclair/typebox/-/typebox-0.27.8.tgz#6667fac16c436b5434a387a34dedb013198f6e6e"
  integrity sha512-+Fj43pSMwJs4KRrH/938Uf+uAELIgVBmQzg/q1YG10djyfA3TnrU8N8XzqCh/okZdszqBQTZf96idMfE5lnwTA==

"@sinonjs/commons@^3.0.0":
  version "3.0.1"
  resolved "https://registry.yarnpkg.com/@sinonjs/commons/-/commons-3.0.1.tgz#1029357e44ca901a615585f6d27738dbc89084cd"
  integrity sha512-K3mCHKQ9sVh8o1C9cxkwxaOmXoAMlDxC1mYyHrjqOWEcBjYr76t96zL2zlj5dUGZ3HSw240X1qgH3Mjf1yJWpQ==
  dependencies:
    type-detect "4.0.8"

"@sinonjs/fake-timers@^10.0.2":
  version "10.3.0"
  resolved "https://registry.yarnpkg.com/@sinonjs/fake-timers/-/fake-timers-10.3.0.tgz#55fdff1ecab9f354019129daf4df0dd4d923ea66"
  integrity sha512-V4BG07kuYSUkTCSBHG8G8TNhM+F19jXFWnQtzj+we8DrkpSBCee9Z3Ms8yiGer/dlmhe35/Xdgyo3/0rQKg7YA==
  dependencies:
    "@sinonjs/commons" "^3.0.0"

"@tsconfig/recommended@^1.0.7":
  version "1.0.7"
  resolved "https://registry.yarnpkg.com/@tsconfig/recommended/-/recommended-1.0.7.tgz#fdd95fc2c8d643c8b4a8ca45fd68eea248512407"
  integrity sha512-xiNMgCuoy4mCL4JTywk9XFs5xpRUcKxtWEcMR6FNMtsgewYTIgIR+nvlP4A4iRCAzRsHMnPhvTRrzp4AGcRTEA==

"@types/babel__core@^7.1.14":
  version "7.20.5"
  resolved "https://registry.yarnpkg.com/@types/babel__core/-/babel__core-7.20.5.tgz#3df15f27ba85319caa07ba08d0721889bb39c017"
  integrity sha512-qoQprZvz5wQFJwMDqeseRXWv3rqMvhgpbXFfVyWhbx9X47POIA6i/+dXefEmZKoAgOaTdaIgNSMqMIU61yRyzA==
  dependencies:
    "@babel/parser" "^7.20.7"
    "@babel/types" "^7.20.7"
    "@types/babel__generator" "*"
    "@types/babel__template" "*"
    "@types/babel__traverse" "*"

"@types/babel__generator@*":
  version "7.6.8"
  resolved "https://registry.yarnpkg.com/@types/babel__generator/-/babel__generator-7.6.8.tgz#f836c61f48b1346e7d2b0d93c6dacc5b9535d3ab"
  integrity sha512-ASsj+tpEDsEiFr1arWrlN6V3mdfjRMZt6LtK/Vp/kreFLnr5QH5+DhvD5nINYZXzwJvXeGq+05iUXcAzVrqWtw==
  dependencies:
    "@babel/types" "^7.0.0"

"@types/babel__template@*":
  version "7.4.4"
  resolved "https://registry.yarnpkg.com/@types/babel__template/-/babel__template-7.4.4.tgz#5672513701c1b2199bc6dad636a9d7491586766f"
  integrity sha512-h/NUaSyG5EyxBIp8YRxo4RMe2/qQgvyowRwVMzhYhBCONbW8PUsg4lkFMrhgZhUe5z3L3MiLDuvyJ/CaPa2A8A==
  dependencies:
    "@babel/parser" "^7.1.0"
    "@babel/types" "^7.0.0"

"@types/babel__traverse@*", "@types/babel__traverse@^7.0.6":
  version "7.20.6"
  resolved "https://registry.yarnpkg.com/@types/babel__traverse/-/babel__traverse-7.20.6.tgz#8dc9f0ae0f202c08d8d4dab648912c8d6038e3f7"
  integrity sha512-r1bzfrm0tomOI8g1SzvCaQHo6Lcv6zu0EA+W2kHrt8dyrHQxGzBBL4kdkzIS+jBMV+EYcMAEAqXqYaLJq5rOZg==
  dependencies:
    "@babel/types" "^7.20.7"

"@types/graceful-fs@^4.1.3":
  version "4.1.9"
  resolved "https://registry.yarnpkg.com/@types/graceful-fs/-/graceful-fs-4.1.9.tgz#2a06bc0f68a20ab37b3e36aa238be6abdf49e8b4"
  integrity sha512-olP3sd1qOEe5dXTSaFvQG+02VdRXcdytWLAZsAq1PecU8uqQAhkrnbli7DagjtXKW/Bl7YJbUsa8MPcuc8LHEQ==
  dependencies:
    "@types/node" "*"

"@types/istanbul-lib-coverage@*", "@types/istanbul-lib-coverage@^2.0.0", "@types/istanbul-lib-coverage@^2.0.1":
  version "2.0.6"
  resolved "https://registry.yarnpkg.com/@types/istanbul-lib-coverage/-/istanbul-lib-coverage-2.0.6.tgz#7739c232a1fee9b4d3ce8985f314c0c6d33549d7"
  integrity sha512-2QF/t/auWm0lsy8XtKVPG19v3sSOQlJe/YHZgfjb/KBBHOGSV+J2q/S671rcq9uTBrLAXmZpqJiaQbMT+zNU1w==

"@types/istanbul-lib-report@*":
  version "3.0.3"
  resolved "https://registry.yarnpkg.com/@types/istanbul-lib-report/-/istanbul-lib-report-3.0.3.tgz#53047614ae72e19fc0401d872de3ae2b4ce350bf"
  integrity sha512-NQn7AHQnk/RSLOxrBbGyJM/aVQ+pjj5HCgasFxc0K/KhoATfQ/47AyUl15I2yBUpihjmas+a+VJBOqecrFH+uA==
  dependencies:
    "@types/istanbul-lib-coverage" "*"

"@types/istanbul-reports@^3.0.0":
  version "3.0.4"
  resolved "https://registry.yarnpkg.com/@types/istanbul-reports/-/istanbul-reports-3.0.4.tgz#0f03e3d2f670fbdac586e34b433783070cc16f54"
  integrity sha512-pk2B1NWalF9toCRu6gjBzR69syFjP4Od8WRAX+0mmf9lAjCRicLOWc+ZrxZHx/0XRjotgkF9t6iaMJ+aXcOdZQ==
  dependencies:
    "@types/istanbul-lib-report" "*"

"@types/jest@^29.5.0":
  version "29.5.12"
  resolved "https://registry.yarnpkg.com/@types/jest/-/jest-29.5.12.tgz#7f7dc6eb4cf246d2474ed78744b05d06ce025544"
  integrity sha512-eDC8bTvT/QhYdxJAulQikueigY5AsdBRH2yDKW3yveW7svY3+DzN84/2NUgkw10RTiJbWqZrTtoGVdYlvFJdLw==
  dependencies:
    expect "^29.0.0"
    pretty-format "^29.0.0"

"@types/json-schema@^7.0.15", "@types/json-schema@^7.0.9":
  version "7.0.15"
  resolved "https://registry.yarnpkg.com/@types/json-schema/-/json-schema-7.0.15.tgz#596a1747233694d50f6ad8a7869fcb6f56cf5841"
  integrity sha512-5+fP8P8MFNC+AyZCDxrB2pkZFPGzqQWUzpSeuuVLvm8VMcorNYavBqoFcxK8bQz4Qsbn4oUEEem4wDLfcysGHA==

"@types/json5@^0.0.29":
  version "0.0.29"
  resolved "https://registry.yarnpkg.com/@types/json5/-/json5-0.0.29.tgz#ee28707ae94e11d2b827bcbe5270bcea7f3e71ee"
  integrity sha512-dRLjCWHYg4oaA77cxO64oO+7JwCwnIzkZPdrrC71jQmQtlhM556pwKo5bUzqvZndkVbeFLIIi+9TC40JNF5hNQ==

"@types/node@*":
  version "22.4.1"
  resolved "https://registry.yarnpkg.com/@types/node/-/node-22.4.1.tgz#9b595d292c65b94c20923159e2ce947731b6fdce"
  integrity sha512-1tbpb9325+gPnKK0dMm+/LMriX0vKxf6RnB0SZUqfyVkQ4fMgUSySqhxE/y8Jvs4NyF1yHzTfG9KlnkIODxPKg==
  dependencies:
    undici-types "~6.19.2"

"@types/retry@0.12.0":
  version "0.12.0"
  resolved "https://registry.yarnpkg.com/@types/retry/-/retry-0.12.0.tgz#2b35eccfcee7d38cd72ad99232fbd58bffb3c84d"
  integrity sha512-wWKOClTTiizcZhXnPY4wikVAwmdYHp8q6DmC+EJUzAMsycb7HB32Kh9RN4+0gExjmPmZSAQjgURXIGATPegAvA==

"@types/semver@^7.3.12":
  version "7.5.8"
  resolved "https://registry.yarnpkg.com/@types/semver/-/semver-7.5.8.tgz#8268a8c57a3e4abd25c165ecd36237db7948a55e"
  integrity sha512-I8EUhyrgfLrcTkzV3TSsGyl1tSuPrEDzr0yd5m90UgNxQkyDXULk3b6MlQqTCpZpNtWe1K0hzclnZkTcLBe2UQ==

"@types/stack-utils@^2.0.0":
  version "2.0.3"
  resolved "https://registry.yarnpkg.com/@types/stack-utils/-/stack-utils-2.0.3.tgz#6209321eb2c1712a7e7466422b8cb1fc0d9dd5d8"
  integrity sha512-9aEbYZ3TbYMznPdcdr3SmIrLXwC/AKZXQeCf9Pgao5CKb8CyHuEX5jzWPTkvregvhRJHcpRO6BFoGW9ycaOkYw==

"@types/uuid@^10.0.0":
  version "10.0.0"
  resolved "https://registry.yarnpkg.com/@types/uuid/-/uuid-10.0.0.tgz#e9c07fe50da0f53dc24970cca94d619ff03f6f6d"
  integrity sha512-7gqG38EyHgyP1S+7+xomFtL+ZNHcKv6DwNaCZmJmo1vgMugyF3TCnXVg4t1uk89mLNwnLtnY3TpOpCOyp1/xHQ==

"@types/yargs-parser@*":
  version "21.0.3"
  resolved "https://registry.yarnpkg.com/@types/yargs-parser/-/yargs-parser-21.0.3.tgz#815e30b786d2e8f0dcd85fd5bcf5e1a04d008f15"
  integrity sha512-I4q9QU9MQv4oEOz4tAHJtNz1cwuLxn2F3xcc2iV5WdqLPpUnj30aUuxt1mAxYTG+oe8CZMV/+6rU4S4gRDzqtQ==

"@types/yargs@^17.0.8":
  version "17.0.33"
  resolved "https://registry.yarnpkg.com/@types/yargs/-/yargs-17.0.33.tgz#8c32303da83eec050a84b3c7ae7b9f922d13e32d"
  integrity sha512-WpxBCKWPLr4xSsHgz511rFJAM+wS28w2zEO1QDNY5zM/S8ok70NNfztH0xwhqKyaK0OHCbN98LDAZuy1ctxDkA==
  dependencies:
    "@types/yargs-parser" "*"

"@typescript-eslint/eslint-plugin@^5.59.8":
  version "5.62.0"
  resolved "https://registry.yarnpkg.com/@typescript-eslint/eslint-plugin/-/eslint-plugin-5.62.0.tgz#aeef0328d172b9e37d9bab6dbc13b87ed88977db"
  integrity sha512-TiZzBSJja/LbhNPvk6yc0JrX9XqhQ0hdh6M2svYfsHGejaKFIAGd9MQ+ERIMzLGlN/kZoYIgdxFV0PuljTKXag==
  dependencies:
    "@eslint-community/regexpp" "^4.4.0"
    "@typescript-eslint/scope-manager" "5.62.0"
    "@typescript-eslint/type-utils" "5.62.0"
    "@typescript-eslint/utils" "5.62.0"
    debug "^4.3.4"
    graphemer "^1.4.0"
    ignore "^5.2.0"
    natural-compare-lite "^1.4.0"
    semver "^7.3.7"
    tsutils "^3.21.0"

"@typescript-eslint/parser@^5.59.8":
  version "5.62.0"
  resolved "https://registry.yarnpkg.com/@typescript-eslint/parser/-/parser-5.62.0.tgz#1b63d082d849a2fcae8a569248fbe2ee1b8a56c7"
  integrity sha512-VlJEV0fOQ7BExOsHYAGrgbEiZoi8D+Bl2+f6V2RrXerRSylnp+ZBHmPvaIa8cz0Ajx7WO7Z5RqfgYg7ED1nRhA==
  dependencies:
    "@typescript-eslint/scope-manager" "5.62.0"
    "@typescript-eslint/types" "5.62.0"
    "@typescript-eslint/typescript-estree" "5.62.0"
    debug "^4.3.4"

"@typescript-eslint/scope-manager@5.62.0":
  version "5.62.0"
  resolved "https://registry.yarnpkg.com/@typescript-eslint/scope-manager/-/scope-manager-5.62.0.tgz#d9457ccc6a0b8d6b37d0eb252a23022478c5460c"
  integrity sha512-VXuvVvZeQCQb5Zgf4HAxc04q5j+WrNAtNh9OwCsCgpKqESMTu3tF/jhZ3xG6T4NZwWl65Bg8KuS2uEvhSfLl0w==
  dependencies:
    "@typescript-eslint/types" "5.62.0"
    "@typescript-eslint/visitor-keys" "5.62.0"

"@typescript-eslint/type-utils@5.62.0":
  version "5.62.0"
  resolved "https://registry.yarnpkg.com/@typescript-eslint/type-utils/-/type-utils-5.62.0.tgz#286f0389c41681376cdad96b309cedd17d70346a"
  integrity sha512-xsSQreu+VnfbqQpW5vnCJdq1Z3Q0U31qiWmRhr98ONQmcp/yhiPJFPq8MXiJVLiksmOKSjIldZzkebzHuCGzew==
  dependencies:
    "@typescript-eslint/typescript-estree" "5.62.0"
    "@typescript-eslint/utils" "5.62.0"
    debug "^4.3.4"
    tsutils "^3.21.0"

"@typescript-eslint/types@5.62.0":
  version "5.62.0"
  resolved "https://registry.yarnpkg.com/@typescript-eslint/types/-/types-5.62.0.tgz#258607e60effa309f067608931c3df6fed41fd2f"
  integrity sha512-87NVngcbVXUahrRTqIK27gD2t5Cu1yuCXxbLcFtCzZGlfyVWWh8mLHkoxzjsB6DDNnvdL+fW8MiwPEJyGJQDgQ==

"@typescript-eslint/typescript-estree@5.62.0":
  version "5.62.0"
  resolved "https://registry.yarnpkg.com/@typescript-eslint/typescript-estree/-/typescript-estree-5.62.0.tgz#7d17794b77fabcac615d6a48fb143330d962eb9b"
  integrity sha512-CmcQ6uY7b9y694lKdRB8FEel7JbU/40iSAPomu++SjLMntB+2Leay2LO6i8VnJk58MtE9/nQSFIH6jpyRWyYzA==
  dependencies:
    "@typescript-eslint/types" "5.62.0"
    "@typescript-eslint/visitor-keys" "5.62.0"
    debug "^4.3.4"
    globby "^11.1.0"
    is-glob "^4.0.3"
    semver "^7.3.7"
    tsutils "^3.21.0"

"@typescript-eslint/utils@5.62.0":
  version "5.62.0"
  resolved "https://registry.yarnpkg.com/@typescript-eslint/utils/-/utils-5.62.0.tgz#141e809c71636e4a75daa39faed2fb5f4b10df86"
  integrity sha512-n8oxjeb5aIbPFEtmQxQYOLI0i9n5ySBEY/ZEHHZqKQSFnxio1rv6dthascc9dLuwrL0RC5mPCxB7vnAVGAYWAQ==
  dependencies:
    "@eslint-community/eslint-utils" "^4.2.0"
    "@types/json-schema" "^7.0.9"
    "@types/semver" "^7.3.12"
    "@typescript-eslint/scope-manager" "5.62.0"
    "@typescript-eslint/types" "5.62.0"
    "@typescript-eslint/typescript-estree" "5.62.0"
    eslint-scope "^5.1.1"
    semver "^7.3.7"

"@typescript-eslint/visitor-keys@5.62.0":
  version "5.62.0"
  resolved "https://registry.yarnpkg.com/@typescript-eslint/visitor-keys/-/visitor-keys-5.62.0.tgz#2174011917ce582875954ffe2f6912d5931e353e"
  integrity sha512-07ny+LHRzQXepkGg6w0mFY41fVUNBrL2Roj/++7V1txKugfjm/Ci/qSND03r2RhlJhJYMcTn9AhhSSqQp0Ysyw==
  dependencies:
    "@typescript-eslint/types" "5.62.0"
    eslint-visitor-keys "^3.3.0"

"@ungap/structured-clone@^1.2.0":
  version "1.2.0"
  resolved "https://registry.yarnpkg.com/@ungap/structured-clone/-/structured-clone-1.2.0.tgz#756641adb587851b5ccb3e095daf27ae581c8406"
  integrity sha512-zuVdFrMJiuCDQUMCzQaD6KL28MjnqqN8XnAqiEq9PNm/hCPTSGfrXCOfwj1ow4LFb/tNymJPwsNbVePc1xFqrQ==

acorn-jsx@^5.3.2:
  version "5.3.2"
  resolved "https://registry.yarnpkg.com/acorn-jsx/-/acorn-jsx-5.3.2.tgz#7ed5bb55908b3b2f1bc55c6af1653bada7f07937"
  integrity sha512-rq9s+JNhf0IChjtDXxllJ7g41oZk5SlXtp0LHwyA5cejwn7vKmKp4pPri6YEePv2PU65sAsegbXtIinmDFDXgQ==

acorn@^8.12.0, acorn@^8.9.0:
  version "8.12.1"
  resolved "https://registry.yarnpkg.com/acorn/-/acorn-8.12.1.tgz#71616bdccbe25e27a54439e0046e89ca76df2248"
  integrity sha512-tcpGyI9zbizT9JbV6oYE477V6mTlXvvi0T0G3SNIYE2apm/G5huBa1+K89VGeovbg+jycCrfhl3ADxErOuO6Jg==

ajv@^6.12.4:
  version "6.12.6"
  resolved "https://registry.yarnpkg.com/ajv/-/ajv-6.12.6.tgz#baf5a62e802b07d977034586f8c3baf5adf26df4"
  integrity sha512-j3fVLgvTo527anyYyJOGTYJbG+vnnQYvE0m5mmkc1TK+nxAppkCLMIL0aZ4dblVCNoGShhm+kzE4ZUykBoMg4g==
  dependencies:
    fast-deep-equal "^3.1.1"
    fast-json-stable-stringify "^2.0.0"
    json-schema-traverse "^0.4.1"
    uri-js "^4.2.2"

ansi-escapes@^4.2.1:
  version "4.3.2"
  resolved "https://registry.yarnpkg.com/ansi-escapes/-/ansi-escapes-4.3.2.tgz#6b2291d1db7d98b6521d5f1efa42d0f3a9feb65e"
  integrity sha512-gKXj5ALrKWQLsYG9jlTRmR/xKluxHV+Z9QEwNIgCfM1/uwPMCuzVVnh5mwTd+OuBZcwSIMbqssNWRm1lE51QaQ==
  dependencies:
    type-fest "^0.21.3"

ansi-regex@^5.0.1:
  version "5.0.1"
  resolved "https://registry.yarnpkg.com/ansi-regex/-/ansi-regex-5.0.1.tgz#082cb2c89c9fe8659a311a53bd6a4dc5301db304"
  integrity sha512-quJQXlTSUGL2LH9SUXo8VwsY4soanhgo6LNSm84E1LBcE8s3O0wpdiRzyR9z/ZZJMlMWv37qOOb9pdJlMUEKFQ==

ansi-styles@^3.2.1:
  version "3.2.1"
  resolved "https://registry.yarnpkg.com/ansi-styles/-/ansi-styles-3.2.1.tgz#41fbb20243e50b12be0f04b8dedbf07520ce841d"
  integrity sha512-VT0ZI6kZRdTh8YyJw3SMbYm/u+NqfsAxEpWO0Pf9sq8/e94WxxOpPKx9FR1FlyCtOVDNOQ+8ntlqFxiRc+r5qA==
  dependencies:
    color-convert "^1.9.0"

ansi-styles@^4.0.0, ansi-styles@^4.1.0:
  version "4.3.0"
  resolved "https://registry.yarnpkg.com/ansi-styles/-/ansi-styles-4.3.0.tgz#edd803628ae71c04c85ae7a0906edad34b648937"
  integrity sha512-zbB9rCJAT1rbjiVDb2hqKFHNYLxgtk8NURxZ3IZwD3F6NtxbXZQCnnSi1Lkx+IDohdPlFp222wVALIheZJQSEg==
  dependencies:
    color-convert "^2.0.1"

ansi-styles@^5.0.0:
  version "5.2.0"
  resolved "https://registry.yarnpkg.com/ansi-styles/-/ansi-styles-5.2.0.tgz#07449690ad45777d1924ac2abb2fc8895dba836b"
  integrity sha512-Cxwpt2SfTzTtXcfOlzGEee8O+c+MmUgGrNiBcXnuWxuFJHe6a5Hz7qwhwe5OgaSYI0IJvkLqWX1ASG+cJOkEiA==

anymatch@^3.0.3:
  version "3.1.3"
  resolved "https://registry.yarnpkg.com/anymatch/-/anymatch-3.1.3.tgz#790c58b19ba1720a84205b57c618d5ad8524973e"
  integrity sha512-KMReFUr0B4t+D+OBkjR3KYqvocp2XaSzO55UcB6mgQMd3KbcE+mWTyvVV7D/zsdEbNnV6acZUutkiHQXvTr1Rw==
  dependencies:
    normalize-path "^3.0.0"
    picomatch "^2.0.4"

argparse@^1.0.7:
  version "1.0.10"
  resolved "https://registry.yarnpkg.com/argparse/-/argparse-1.0.10.tgz#bcd6791ea5ae09725e17e5ad988134cd40b3d911"
  integrity sha512-o5Roy6tNG4SL/FOkCAN6RzjiakZS25RLYFrcMttJqbdd8BWrnA+fGz57iN5Pb06pvBGvl5gQ0B48dJlslXvoTg==
  dependencies:
    sprintf-js "~1.0.2"

argparse@^2.0.1:
  version "2.0.1"
  resolved "https://registry.yarnpkg.com/argparse/-/argparse-2.0.1.tgz#246f50f3ca78a3240f6c997e8a9bd1eac49e4b38"
  integrity sha512-8+9WqebbFzpX9OR+Wa6O29asIogeRMzcGtAINdpMHHyAg10f05aSFVBbcEqGf/PXw1EjAZ+q2/bEBg3DvurK3Q==

array-buffer-byte-length@^1.0.1:
  version "1.0.1"
  resolved "https://registry.yarnpkg.com/array-buffer-byte-length/-/array-buffer-byte-length-1.0.1.tgz#1e5583ec16763540a27ae52eed99ff899223568f"
  integrity sha512-ahC5W1xgou+KTXix4sAO8Ki12Q+jf4i0+tmk3sC+zgcynshkHxzpXdImBehiUYKKKDwvfFiJl1tZt6ewscS1Mg==
  dependencies:
    call-bind "^1.0.5"
    is-array-buffer "^3.0.4"

array-includes@^3.1.7:
  version "3.1.8"
  resolved "https://registry.yarnpkg.com/array-includes/-/array-includes-3.1.8.tgz#5e370cbe172fdd5dd6530c1d4aadda25281ba97d"
  integrity sha512-itaWrbYbqpGXkGhZPGUulwnhVf5Hpy1xiCFsGqyIGglbBxmG5vSjxQen3/WGOjPpNEv1RtBLKxbmVXm8HpJStQ==
  dependencies:
    call-bind "^1.0.7"
    define-properties "^1.2.1"
    es-abstract "^1.23.2"
    es-object-atoms "^1.0.0"
    get-intrinsic "^1.2.4"
    is-string "^1.0.7"

array-union@^2.1.0:
  version "2.1.0"
  resolved "https://registry.yarnpkg.com/array-union/-/array-union-2.1.0.tgz#b798420adbeb1de828d84acd8a2e23d3efe85e8d"
  integrity sha512-HGyxoOTYUyCM6stUe6EJgnd4EoewAI7zMdfqO+kGjnlZmBDz/cR5pf8r/cR4Wq60sL/p0IkcjUEEPwS3GFrIyw==

array.prototype.findlastindex@^1.2.3:
  version "1.2.5"
  resolved "https://registry.yarnpkg.com/array.prototype.findlastindex/-/array.prototype.findlastindex-1.2.5.tgz#8c35a755c72908719453f87145ca011e39334d0d"
  integrity sha512-zfETvRFA8o7EiNn++N5f/kaCw221hrpGsDmcpndVupkPzEc1Wuf3VgC0qby1BbHs7f5DVYjgtEU2LLh5bqeGfQ==
  dependencies:
    call-bind "^1.0.7"
    define-properties "^1.2.1"
    es-abstract "^1.23.2"
    es-errors "^1.3.0"
    es-object-atoms "^1.0.0"
    es-shim-unscopables "^1.0.2"

array.prototype.flat@^1.3.2:
  version "1.3.2"
  resolved "https://registry.yarnpkg.com/array.prototype.flat/-/array.prototype.flat-1.3.2.tgz#1476217df8cff17d72ee8f3ba06738db5b387d18"
  integrity sha512-djYB+Zx2vLewY8RWlNCUdHjDXs2XOgm602S9E7P/UpHgfeHL00cRiIF+IN/G/aUJ7kGPb6yO/ErDI5V2s8iycA==
  dependencies:
    call-bind "^1.0.2"
    define-properties "^1.2.0"
    es-abstract "^1.22.1"
    es-shim-unscopables "^1.0.0"

array.prototype.flatmap@^1.3.2:
  version "1.3.2"
  resolved "https://registry.yarnpkg.com/array.prototype.flatmap/-/array.prototype.flatmap-1.3.2.tgz#c9a7c6831db8e719d6ce639190146c24bbd3e527"
  integrity sha512-Ewyx0c9PmpcsByhSW4r+9zDU7sGjFc86qf/kKtuSCRdhfbk0SNLLkaT5qvcHnRGgc5NP/ly/y+qkXkqONX54CQ==
  dependencies:
    call-bind "^1.0.2"
    define-properties "^1.2.0"
    es-abstract "^1.22.1"
    es-shim-unscopables "^1.0.0"

arraybuffer.prototype.slice@^1.0.3:
  version "1.0.3"
  resolved "https://registry.yarnpkg.com/arraybuffer.prototype.slice/-/arraybuffer.prototype.slice-1.0.3.tgz#097972f4255e41bc3425e37dc3f6421cf9aefde6"
  integrity sha512-bMxMKAjg13EBSVscxTaYA4mRc5t1UAXa2kXiGTNfZ079HIWXEkKmkgFrh/nJqamaLSrXO5H4WFFkPEaLJWbs3A==
  dependencies:
    array-buffer-byte-length "^1.0.1"
    call-bind "^1.0.5"
    define-properties "^1.2.1"
    es-abstract "^1.22.3"
    es-errors "^1.2.1"
    get-intrinsic "^1.2.3"
    is-array-buffer "^3.0.4"
    is-shared-array-buffer "^1.0.2"

async@^3.2.3:
  version "3.2.6"
  resolved "https://registry.yarnpkg.com/async/-/async-3.2.6.tgz#1b0728e14929d51b85b449b7f06e27c1145e38ce"
  integrity sha512-htCUDlxyyCLMgaM3xXg0C0LW2xqfuQ6p05pCEIsXuyQ+a1koYKTuBMzRNwmybfLgvJDMd0r1LTn4+E0Ti6C2AA==

available-typed-arrays@^1.0.7:
  version "1.0.7"
  resolved "https://registry.yarnpkg.com/available-typed-arrays/-/available-typed-arrays-1.0.7.tgz#a5cc375d6a03c2efc87a553f3e0b1522def14846"
  integrity sha512-wvUjBtSGN7+7SjNpq/9M2Tg350UZD3q62IFZLbRAR1bSMlCo1ZaeW+BJ+D090e4hIIZLBcTDWe4Mh4jvUDajzQ==
  dependencies:
    possible-typed-array-names "^1.0.0"

babel-jest@^29.7.0:
  version "29.7.0"
  resolved "https://registry.yarnpkg.com/babel-jest/-/babel-jest-29.7.0.tgz#f4369919225b684c56085998ac63dbd05be020d5"
  integrity sha512-BrvGY3xZSwEcCzKvKsCi2GgHqDqsYkOP4/by5xCgIwGXQxIEh+8ew3gmrE1y7XRR6LHZIj6yLYnUi/mm2KXKBg==
  dependencies:
    "@jest/transform" "^29.7.0"
    "@types/babel__core" "^7.1.14"
    babel-plugin-istanbul "^6.1.1"
    babel-preset-jest "^29.6.3"
    chalk "^4.0.0"
    graceful-fs "^4.2.9"
    slash "^3.0.0"

babel-plugin-istanbul@^6.1.1:
  version "6.1.1"
  resolved "https://registry.yarnpkg.com/babel-plugin-istanbul/-/babel-plugin-istanbul-6.1.1.tgz#fa88ec59232fd9b4e36dbbc540a8ec9a9b47da73"
  integrity sha512-Y1IQok9821cC9onCx5otgFfRm7Lm+I+wwxOx738M/WLPZ9Q42m4IG5W0FNX8WLL2gYMZo3JkuXIH2DOpWM+qwA==
  dependencies:
    "@babel/helper-plugin-utils" "^7.0.0"
    "@istanbuljs/load-nyc-config" "^1.0.0"
    "@istanbuljs/schema" "^0.1.2"
    istanbul-lib-instrument "^5.0.4"
    test-exclude "^6.0.0"

babel-plugin-jest-hoist@^29.6.3:
  version "29.6.3"
  resolved "https://registry.yarnpkg.com/babel-plugin-jest-hoist/-/babel-plugin-jest-hoist-29.6.3.tgz#aadbe943464182a8922c3c927c3067ff40d24626"
  integrity sha512-ESAc/RJvGTFEzRwOTT4+lNDk/GNHMkKbNzsvT0qKRfDyyYTskxB5rnU2njIDYVxXCBHHEI1c0YwHob3WaYujOg==
  dependencies:
    "@babel/template" "^7.3.3"
    "@babel/types" "^7.3.3"
    "@types/babel__core" "^7.1.14"
    "@types/babel__traverse" "^7.0.6"

babel-preset-current-node-syntax@^1.0.0:
  version "1.1.0"
  resolved "https://registry.yarnpkg.com/babel-preset-current-node-syntax/-/babel-preset-current-node-syntax-1.1.0.tgz#9a929eafece419612ef4ae4f60b1862ebad8ef30"
  integrity sha512-ldYss8SbBlWva1bs28q78Ju5Zq1F+8BrqBZZ0VFhLBvhh6lCpC2o3gDJi/5DRLs9FgYZCnmPYIVFU4lRXCkyUw==
  dependencies:
    "@babel/plugin-syntax-async-generators" "^7.8.4"
    "@babel/plugin-syntax-bigint" "^7.8.3"
    "@babel/plugin-syntax-class-properties" "^7.12.13"
    "@babel/plugin-syntax-class-static-block" "^7.14.5"
    "@babel/plugin-syntax-import-attributes" "^7.24.7"
    "@babel/plugin-syntax-import-meta" "^7.10.4"
    "@babel/plugin-syntax-json-strings" "^7.8.3"
    "@babel/plugin-syntax-logical-assignment-operators" "^7.10.4"
    "@babel/plugin-syntax-nullish-coalescing-operator" "^7.8.3"
    "@babel/plugin-syntax-numeric-separator" "^7.10.4"
    "@babel/plugin-syntax-object-rest-spread" "^7.8.3"
    "@babel/plugin-syntax-optional-catch-binding" "^7.8.3"
    "@babel/plugin-syntax-optional-chaining" "^7.8.3"
    "@babel/plugin-syntax-private-property-in-object" "^7.14.5"
    "@babel/plugin-syntax-top-level-await" "^7.14.5"

babel-preset-jest@^29.6.3:
  version "29.6.3"
  resolved "https://registry.yarnpkg.com/babel-preset-jest/-/babel-preset-jest-29.6.3.tgz#fa05fa510e7d493896d7b0dd2033601c840f171c"
  integrity sha512-0B3bhxR6snWXJZtR/RliHTDPRgn1sNHOR0yVtq/IiQFyuOVjFS+wuio/R4gSNkyYmKmJB4wGZv2NZanmKmTnNA==
  dependencies:
    babel-plugin-jest-hoist "^29.6.3"
    babel-preset-current-node-syntax "^1.0.0"

balanced-match@^1.0.0:
  version "1.0.2"
  resolved "https://registry.yarnpkg.com/balanced-match/-/balanced-match-1.0.2.tgz#e83e3a7e3f300b34cb9d87f615fa0cbf357690ee"
  integrity sha512-3oSeUO0TMV67hN1AmbXsK4yaqU7tjiHlbxRDZOpH0KW9+CeX4bRAaX0Anxt0tx2MrpRpWwQaPwIlISEJhYU5Pw==

base64-js@^1.5.1:
  version "1.5.1"
  resolved "https://registry.yarnpkg.com/base64-js/-/base64-js-1.5.1.tgz#1b1b440160a5bf7ad40b650f095963481903930a"
  integrity sha512-AKpaYlHn8t4SVbOHCy+b5+KKgvR4vrsD8vbvrbiQJps7fKDTkjkDry6ji0rUJjC0kzbNePLwzxq8iypo41qeWA==

brace-expansion@^1.1.7:
  version "1.1.11"
  resolved "https://registry.yarnpkg.com/brace-expansion/-/brace-expansion-1.1.11.tgz#3c7fcbf529d87226f3d2f52b966ff5271eb441dd"
  integrity sha512-iCuPHDFgrHX7H2vEI/5xpz07zSHB00TpugqhmYtVmMO6518mCuRMoOYFldEBl0g187ufozdaHgWKcYFb61qGiA==
  dependencies:
    balanced-match "^1.0.0"
    concat-map "0.0.1"

brace-expansion@^2.0.1:
  version "2.0.1"
  resolved "https://registry.yarnpkg.com/brace-expansion/-/brace-expansion-2.0.1.tgz#1edc459e0f0c548486ecf9fc99f2221364b9a0ae"
  integrity sha512-XnAIvQ8eM+kC6aULx6wuQiwVsnzsi9d3WxzV3FpWTGA19F621kwdbsAcFKXgKUHZWsy+mY6iL1sHTxWEFCytDA==
  dependencies:
    balanced-match "^1.0.0"

braces@^3.0.3:
  version "3.0.3"
  resolved "https://registry.yarnpkg.com/braces/-/braces-3.0.3.tgz#490332f40919452272d55a8480adc0c441358789"
  integrity sha512-yQbXgO/OSZVD2IsiLlro+7Hf6Q18EJrKSEsdoMzKePKXct3gvD8oLcOQdIzGupr5Fj+EDe8gO/lxc1BzfMpxvA==
  dependencies:
    fill-range "^7.1.1"

browserslist@^4.23.1:
  version "4.23.3"
  resolved "https://registry.yarnpkg.com/browserslist/-/browserslist-4.23.3.tgz#debb029d3c93ebc97ffbc8d9cbb03403e227c800"
  integrity sha512-btwCFJVjI4YWDNfau8RhZ+B1Q/VLoUITrm3RlP6y1tYGWIOa+InuYiRGXUBXo8nA1qKmHMyLB/iVQg5TT4eFoA==
  dependencies:
    caniuse-lite "^1.0.30001646"
    electron-to-chromium "^1.5.4"
    node-releases "^2.0.18"
    update-browserslist-db "^1.1.0"

bs-logger@0.x:
  version "0.2.6"
  resolved "https://registry.yarnpkg.com/bs-logger/-/bs-logger-0.2.6.tgz#eb7d365307a72cf974cc6cda76b68354ad336bd8"
  integrity sha512-pd8DCoxmbgc7hyPKOvxtqNcjYoOsABPQdcCUjGp3d42VR2CX1ORhk2A87oqqu5R1kk+76nsxZupkmyd+MVtCog==
  dependencies:
    fast-json-stable-stringify "2.x"

bser@2.1.1:
  version "2.1.1"
  resolved "https://registry.yarnpkg.com/bser/-/bser-2.1.1.tgz#e6787da20ece9d07998533cfd9de6f5c38f4bc05"
  integrity sha512-gQxTNE/GAfIIrmHLUE3oJyp5FO6HRBfhjnw4/wMmA63ZGDJnWBmgY/lyQBpnDUkGmAhbSe39tx2d/iTOAfglwQ==
  dependencies:
    node-int64 "^0.4.0"

buffer-from@^1.0.0:
  version "1.1.2"
  resolved "https://registry.yarnpkg.com/buffer-from/-/buffer-from-1.1.2.tgz#2b146a6fd72e80b4f55d255f35ed59a3a9a41bd5"
  integrity sha512-E+XQCRwSbaaiChtv6k6Dwgc+bx+Bs6vuKJHHl5kox/BaKbhiXzqQOwK4cO22yElGp2OCmjwVhT3HmxgyPGnJfQ==

call-bind@^1.0.2, call-bind@^1.0.5, call-bind@^1.0.6, call-bind@^1.0.7:
  version "1.0.7"
  resolved "https://registry.yarnpkg.com/call-bind/-/call-bind-1.0.7.tgz#06016599c40c56498c18769d2730be242b6fa3b9"
  integrity sha512-GHTSNSYICQ7scH7sZ+M2rFopRoLh8t2bLSW6BbgrtLsahOIB5iyAVJf9GjWK3cYTDaMj4XdBpM1cA6pIS0Kv2w==
  dependencies:
    es-define-property "^1.0.0"
    es-errors "^1.3.0"
    function-bind "^1.1.2"
    get-intrinsic "^1.2.4"
    set-function-length "^1.2.1"

callsites@^3.0.0:
  version "3.1.0"
  resolved "https://registry.yarnpkg.com/callsites/-/callsites-3.1.0.tgz#b3630abd8943432f54b3f0519238e33cd7df2f73"
  integrity sha512-P8BjAsXvZS+VIDUI11hHCQEv74YT67YUi5JJFNWIqL235sBmjX4+qx9Muvls5ivyNENctx46xQLQ3aTuE7ssaQ==

camelcase@6, camelcase@^6.2.0:
  version "6.3.0"
  resolved "https://registry.yarnpkg.com/camelcase/-/camelcase-6.3.0.tgz#5685b95eb209ac9c0c177467778c9c84df58ba9a"
  integrity sha512-Gmy6FhYlCY7uOElZUSbxo2UCDH8owEk996gkbrpsgGtrJLM3J7jGxl9Ic7Qwwj4ivOE5AWZWRMecDdF7hqGjFA==

camelcase@^5.3.1:
  version "5.3.1"
  resolved "https://registry.yarnpkg.com/camelcase/-/camelcase-5.3.1.tgz#e3c9b31569e106811df242f715725a1f4c494320"
  integrity sha512-L28STB170nwWS63UjtlEOE3dldQApaJXZkOI1uMFfzf3rRuPegHaHesyee+YxQ+W6SvRDQV6UrdOdRiR153wJg==

caniuse-lite@^1.0.30001646:
  version "1.0.30001651"
  resolved "https://registry.yarnpkg.com/caniuse-lite/-/caniuse-lite-1.0.30001651.tgz#52de59529e8b02b1aedcaaf5c05d9e23c0c28138"
  integrity sha512-9Cf+Xv1jJNe1xPZLGuUXLNkE1BoDkqRqYyFJ9TDYSqhduqA4hu4oR9HluGoWYQC/aj8WHjsGVV+bwkh0+tegRg==

chalk@^2.4.2:
  version "2.4.2"
  resolved "https://registry.yarnpkg.com/chalk/-/chalk-2.4.2.tgz#cd42541677a54333cf541a49108c1432b44c9424"
  integrity sha512-Mti+f9lpJNcwF4tWV8/OrTTtF1gZi+f8FqlyAdouralcFWFQWF2+NgCHShjkCb+IFBLq9buZwE1xckQU4peSuQ==
  dependencies:
    ansi-styles "^3.2.1"
    escape-string-regexp "^1.0.5"
    supports-color "^5.3.0"

chalk@^4.0.0, chalk@^4.0.2:
  version "4.1.2"
  resolved "https://registry.yarnpkg.com/chalk/-/chalk-4.1.2.tgz#aac4e2b7734a740867aeb16bf02aad556a1e7a01"
  integrity sha512-oKnbhFyRIXpUuez8iBMmyEa4nbj4IOQyuhc/wy9kY7/WVPcwIO9VA668Pu8RkO7+0G76SLROeyw9CpQ061i4mA==
  dependencies:
    ansi-styles "^4.1.0"
    supports-color "^7.1.0"

char-regex@^1.0.2:
  version "1.0.2"
  resolved "https://registry.yarnpkg.com/char-regex/-/char-regex-1.0.2.tgz#d744358226217f981ed58f479b1d6bcc29545dcf"
  integrity sha512-kWWXztvZ5SBQV+eRgKFeh8q5sLuZY2+8WUIzlxWVTg+oGwY14qylx1KbKzHd8P6ZYkAg0xyIDU9JMHhyJMZ1jw==

ci-info@^3.2.0:
  version "3.9.0"
  resolved "https://registry.yarnpkg.com/ci-info/-/ci-info-3.9.0.tgz#4279a62028a7b1f262f3473fc9605f5e218c59b4"
  integrity sha512-NIxF55hv4nSqQswkAeiOi1r83xy8JldOFDTWiug55KBu9Jnblncd2U6ViHmYgHf01TPZS77NJBhBMKdWj9HQMQ==

cjs-module-lexer@^1.0.0:
  version "1.3.1"
  resolved "https://registry.yarnpkg.com/cjs-module-lexer/-/cjs-module-lexer-1.3.1.tgz#c485341ae8fd999ca4ee5af2d7a1c9ae01e0099c"
  integrity sha512-a3KdPAANPbNE4ZUv9h6LckSl9zLsYOP4MBmhIPkRaeyybt+r4UghLvq+xw/YwUcC1gqylCkL4rdVs3Lwupjm4Q==

cliui@^8.0.1:
  version "8.0.1"
  resolved "https://registry.yarnpkg.com/cliui/-/cliui-8.0.1.tgz#0c04b075db02cbfe60dc8e6cf2f5486b1a3608aa"
  integrity sha512-BSeNnyus75C4//NQ9gQt1/csTXyo/8Sb+afLAkzAptFuMsod9HFokGNudZpi/oQV73hnVK+sR+5PVRMd+Dr7YQ==
  dependencies:
    string-width "^4.2.0"
    strip-ansi "^6.0.1"
    wrap-ansi "^7.0.0"

co@^4.6.0:
  version "4.6.0"
  resolved "https://registry.yarnpkg.com/co/-/co-4.6.0.tgz#6ea6bdf3d853ae54ccb8e47bfa0bf3f9031fb184"
  integrity sha512-QVb0dM5HvG+uaxitm8wONl7jltx8dqhfU33DcqtOZcLSVIKSDDLDi7+0LbAKiyI8hD9u42m2YxXSkMGWThaecQ==

collect-v8-coverage@^1.0.0:
  version "1.0.2"
  resolved "https://registry.yarnpkg.com/collect-v8-coverage/-/collect-v8-coverage-1.0.2.tgz#c0b29bcd33bcd0779a1344c2136051e6afd3d9e9"
  integrity sha512-lHl4d5/ONEbLlJvaJNtsF/Lz+WvB07u2ycqTYbdrq7UypDXailES4valYb2eWiJFxZlVmpGekfqoxQhzyFdT4Q==

color-convert@^1.9.0:
  version "1.9.3"
  resolved "https://registry.yarnpkg.com/color-convert/-/color-convert-1.9.3.tgz#bb71850690e1f136567de629d2d5471deda4c1e8"
  integrity sha512-QfAUtd+vFdAtFQcC8CCyYt1fYWxSqAiK2cSD6zDB8N3cpsEBAvRxp9zOGg6G/SHHJYAT88/az/IuDGALsNVbGg==
  dependencies:
    color-name "1.1.3"

color-convert@^2.0.1:
  version "2.0.1"
  resolved "https://registry.yarnpkg.com/color-convert/-/color-convert-2.0.1.tgz#72d3a68d598c9bdb3af2ad1e84f21d896abd4de3"
  integrity sha512-RRECPsj7iu/xb5oKYcsFHSppFNnsj/52OVTRKb4zP5onXwVF3zVmmToNcOfGC+CRDpfK/U584fMg38ZHCaElKQ==
  dependencies:
    color-name "~1.1.4"

color-name@1.1.3:
  version "1.1.3"
  resolved "https://registry.yarnpkg.com/color-name/-/color-name-1.1.3.tgz#a7d0558bd89c42f795dd42328f740831ca53bc25"
  integrity sha512-72fSenhMw2HZMTVHeCA9KCmpEIbzWiQsjN+BHcBbS9vr1mtt+vJjPdksIBNUmKAW8TFUDPJK5SUU3QhE9NEXDw==

color-name@~1.1.4:
  version "1.1.4"
  resolved "https://registry.yarnpkg.com/color-name/-/color-name-1.1.4.tgz#c2a09a87acbde69543de6f63fa3995c826c536a2"
  integrity sha512-dOy+3AuW3a2wNbZHIuMZpTcgjGuLU/uBL/ubcZF9OXbDo8ff4O8yVp5Bf0efS8uEoYo5q4Fx7dY9OgQGXgAsQA==

commander@^10.0.1:
  version "10.0.1"
  resolved "https://registry.yarnpkg.com/commander/-/commander-10.0.1.tgz#881ee46b4f77d1c1dccc5823433aa39b022cbe06"
  integrity sha512-y4Mg2tXshplEbSGzx7amzPwKKOCGuoSRP/CjEdwwk0FOGlUbq6lKuoyDZTNZkmxHdJtp54hdfY/JUrdL7Xfdug==

concat-map@0.0.1:
  version "0.0.1"
  resolved "https://registry.yarnpkg.com/concat-map/-/concat-map-0.0.1.tgz#d8a96bd77fd68df7793a73036a3ba0d5405d477b"
  integrity sha512-/Srv4dswyQNBfohGpz9o6Yb3Gz3SrUDqBH5rTuhGR7ahtlbYKnVxw2bCFMRljaA7EXHaXZ8wsHdodFvbkhKmqg==

convert-source-map@^2.0.0:
  version "2.0.0"
  resolved "https://registry.yarnpkg.com/convert-source-map/-/convert-source-map-2.0.0.tgz#4b560f649fc4e918dd0ab75cf4961e8bc882d82a"
  integrity sha512-Kvp459HrV2FEJ1CAsi1Ku+MY3kasH19TFykTz2xWmMeq6bk2NU3XXvfJ+Q61m0xktWwt+1HSYf3JZsTms3aRJg==

create-jest@^29.7.0:
  version "29.7.0"
  resolved "https://registry.yarnpkg.com/create-jest/-/create-jest-29.7.0.tgz#a355c5b3cb1e1af02ba177fe7afd7feee49a5320"
  integrity sha512-Adz2bdH0Vq3F53KEMJOoftQFutWCukm6J24wbPWRO4k1kMY7gS7ds/uoJkNuV8wDCtWWnuwGcJwpWcih+zEW1Q==
  dependencies:
    "@jest/types" "^29.6.3"
    chalk "^4.0.0"
    exit "^0.1.2"
    graceful-fs "^4.2.9"
    jest-config "^29.7.0"
    jest-util "^29.7.0"
    prompts "^2.0.1"

cross-spawn@^7.0.2, cross-spawn@^7.0.3:
  version "7.0.6"
  resolved "https://registry.yarnpkg.com/cross-spawn/-/cross-spawn-7.0.6.tgz#8a58fe78f00dcd70c370451759dfbfaf03e8ee9f"
  integrity sha512-uV2QOWP2nWzsy2aMp8aRibhi9dlzF5Hgh5SHaB9OiTGEyDTiJJyx0uy51QXdyWbtAHNua4XJzUKca3OzKUd3vA==
  dependencies:
    path-key "^3.1.0"
    shebang-command "^2.0.0"
    which "^2.0.1"

data-view-buffer@^1.0.1:
  version "1.0.1"
  resolved "https://registry.yarnpkg.com/data-view-buffer/-/data-view-buffer-1.0.1.tgz#8ea6326efec17a2e42620696e671d7d5a8bc66b2"
  integrity sha512-0lht7OugA5x3iJLOWFhWK/5ehONdprk0ISXqVFn/NFrDu+cuc8iADFrGQz5BnRK7LLU3JmkbXSxaqX+/mXYtUA==
  dependencies:
    call-bind "^1.0.6"
    es-errors "^1.3.0"
    is-data-view "^1.0.1"

data-view-byte-length@^1.0.1:
  version "1.0.1"
  resolved "https://registry.yarnpkg.com/data-view-byte-length/-/data-view-byte-length-1.0.1.tgz#90721ca95ff280677eb793749fce1011347669e2"
  integrity sha512-4J7wRJD3ABAzr8wP+OcIcqq2dlUKp4DVflx++hs5h5ZKydWMI6/D/fAot+yh6g2tHh8fLFTvNOaVN357NvSrOQ==
  dependencies:
    call-bind "^1.0.7"
    es-errors "^1.3.0"
    is-data-view "^1.0.1"

data-view-byte-offset@^1.0.0:
  version "1.0.0"
  resolved "https://registry.yarnpkg.com/data-view-byte-offset/-/data-view-byte-offset-1.0.0.tgz#5e0bbfb4828ed2d1b9b400cd8a7d119bca0ff18a"
  integrity sha512-t/Ygsytq+R995EJ5PZlD4Cu56sWa8InXySaViRzw9apusqsOO2bQP+SbYzAhR0pFKoB+43lYy8rWban9JSuXnA==
  dependencies:
    call-bind "^1.0.6"
    es-errors "^1.3.0"
    is-data-view "^1.0.1"

debug@^3.2.7:
  version "3.2.7"
  resolved "https://registry.yarnpkg.com/debug/-/debug-3.2.7.tgz#72580b7e9145fb39b6676f9c5e5fb100b934179a"
  integrity sha512-CFjzYYAi4ThfiQvizrFQevTTXHtnCqWfe7x1AhgEscTz6ZbLbfoLRLPugTQyBth6f8ZERVUSyWHFD/7Wu4t1XQ==
  dependencies:
    ms "^2.1.1"

debug@^4.1.0, debug@^4.1.1, debug@^4.3.1, debug@^4.3.2, debug@^4.3.4:
  version "4.3.6"
