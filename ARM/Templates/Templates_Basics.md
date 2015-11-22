# Azure Resource Manager Templates Basics
With Resource Manager, you can create a simple template (in JSON format) that defines deployment and configuration of your application. This template is known as a Resource Manager Template and provides a declarative way to define deployment. By using a template, you can repeatedly deploy your application throughout the app lifecycle and have confidence your resources are deployed in a consistent state.

**Template size is limted to 1MB, and each parameter file to 64KB**. The 1 MB limit applies to the final state of the template after it has been expanded with iterative resource definitions, and values for variables and parameters.

The basic syntax of the template is JSON; however, expressions and functions extend the JSON that is available in the template and enable you to create values that are not strict literal values. Expressions are enclosed with brackets ([ and ]), and are evaluated when the template is deployed. Expressions can appear anywhere in a JSON string value and always return another JSON value. If you need to use a literal string that starts with a bracket [, you must use two brackets [[.

## Template Structure
A tempalte consists of 4 basics parts, that constracts the resources needed to deploy the application, the parameters and their valus and any output values.

Basic template structure:
```
{
   "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
   "contentVersion": "",
   "parameters": {  },
   "variables": {  },
   "resources": [  ],
   "outputs": {  }
}
```
|Element | Required | Description |
|--------|----------|-------------|
|$schema | Yes      |location of the JSON schema file that describes the version of the template language|
|contentVersion | Yes | Version of the template (such as 1.0.0.0). When deploying resources using the template, this value can be used to make sure that the right template is being used|
|parameters | No | Values that are provided when deployment is executed to customize resource deployment |
|variables| No | Values that are used as JSON fragments in the template to simplify template language expressions |
|resources | Yes | Types of services that are deployed or updated in a resource group | 
|outputs | No | Values that are returned after deployment |

### Parameters
The parameters section defined the values a user can input while deploying the resources. The values of the parameters can use used thoughout the template.

```
"parameters": {
   "<parameterName>" : {
     "type" : "<type-of-parameter-value>",
     "defaultValue": "<optional-default-value-of-parameter>",
     "allowedValues": [ "<optional-array-of-allowed-values>" ],
     "minValue": <optional-minimum-value-for-int-parameters>,
     "maxValue": <optional-maximum-value-for-int-parameters>,
     "minLength": <optional-minimum-length-for-string-secureString-array-parameters>,
     "maxLength": <optional-maximum-length-for-string-secureString-array-parameters>,
     "metadata": {
         "description": "<optional-description-of-the parameter>" 
     }
   }
}
```
|ELEMENT NAME	|REQUIRED	|DESCRIPTION|
|-------------|---------|-----------|
|parameterName|	Yes	|Name of the parameter. Must be a valid JavaScript identifier.|
|type|	Yes	|Type of the parameter value. See the list below of allowed types.|
|defaultValue|	No	|Default value for the parameter, if no value is provided for the parameter.|
|allowedValues|	No	|Array of allowed values for the parameter to make sure that the right value is provided.|
|minValue|	No	|The minimum value for int type parameters, this value is inclusive.|
|maxValue|	No	|The maximum value for int type parameters, this value is inclusive.|
|minLength|	No	|The minimum length for string, secureString and array type parameters, this value is inclusive.|
|maxLength|	No	|The maximum length for string, secureString and array type parameters, this value is inclusive.|
|description|	No	|Description of the parameter which will be displayed to users of the template through the portal custom template interface.|

**Allowed Values**

* string or secureString - any valid JSON string
* int - any valid JSON integer
* bool - any valid JSON boolean
* object or secureObject - any valid JSON object
* array - any valid JSON array

**Optional Parameter**

To specify a parameter as optional, set its defaultValue to an empty string.

**Secure Parameter Values**

secureString - all passwords, keys, and other secrets should use the **secureString** type. Template parameters with the secureString type cannot be read after resource deployment.

**Parameter File**

Parameters can be passed to the deployment using a parameter file. This is also a JSON file, with the following structure:
```
{
   "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
   "contentVersion": "",
   "parameters": {
    "<parameterName>" : {
      <Parameters structure - same as in template file>
    }
   }
}
```

### Variables
Variables are used to simplify template language expressions. Typically, these variables will be based on values provided from the parameters. Basic variables section structure:
```
"variables": {
   "<variable-name>": "<variable-value>",
   "<variable-name>": { 
       <variable-complex-type-value> 
   }
}
```

Follwing is an example of a variable constructed from two parameters, one that is custructed from other variables and a varialbe that is a complex JSON object:
```
"parameters": {
   "username": {
     "type": "string"
   },
   "password": {
     "type": "secureString"
   },
   "environmentName": {
     "type": "string",
     "allowedValues": [
       "test",
       "prod"
     ]
   }
 },
 "variables": {
   "connectionString": "[concat('Name=', parameters('username'), ';Password=', parameters('password'))]",
   "environmentSettings": {
     "test": {
       "instancesSize": "Small",
       "instancesCount": 1
     },
     "prod": {
       "instancesSize": "Large",
       "instancesCount": 4
     }
   },
   "currentEnvironmentSettings": "[variables('environmentSettings')[parameters('environmentName')]]",
   "instancesSize": "[variables('currentEnvironmentSettings').instancesSize",
   "instancesCount": "[variables('currentEnvironmentSettings').instancesCount"
}
```
### Resources


### Output

## Expressions and functions
## Dependencies
## Linked Templates

## Resources and References

https://azure.microsoft.com/en-us/documentation/articles/resource-group-authoring-templates/

[MVA course](https://mva.microsoft.com/en-us/training-courses/deep-dive-into-azure-resource-manager-scenarios-and-patterns-13793?l=i1m06ZJYB_7001937557)


