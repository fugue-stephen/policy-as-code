# Policy As Code

Ludwig enables you to perform validations on compositions, ensuring they are compliant with your policies and best practices at compile time instead of at runtime.

All organizations have policies that specify how environments should be provisioned in development, QA, and production. This example demonstrates how different validations can be applied to those environments based on an environment variable. A description of the files contained in this repo is below:

<pre>
├── Config                                      Contains configuration information
│   ├── Environment                               Specifies configuration details for each environment
│   │   ├── Dev.lw
│   │   ├── Prod.lw
│   │   └── QA.lw
│   └── Environment.lw                            Checks the environment value to determine which config in Config/Environment to use 
├── Corp                                        Contains compositions for organization-specific policies
│   ├── ArtifactRepo.lw                           Creates an artifact repository (i.e. a place to store executables and files) based on the environment configuration
│   ├── DeveloperEnvironment.lw                   Composition for creating a developer environment
│   ├── DeveloperEnvironmentConfig.lw             userData for creating the developer environment
│   ├── Environment.lw                            Defines the environments where users may run compositions
│   └── InfoSecStandards.lw                       Contains all validations
├── Corp.lw                                     Top-level composition
├── Makefile                                    Makefile for compiling compositions
├── compositions                                Contains compositions that users may run and which must adhere to the validations in `InfoSecStandards.lw`
│   ├── CreateArtifactRepo.lw
│   └── CreateDeveloperEnvironment.lw
</pre>	

The `Environment` type is defined in `Corp/Environment.lw`, which determines acceptable environments in which users may run compositions. The environment is set by an environment variable, `ENVIRONMENT`, and valid choices are `DEV`, `QA`, and `PROD`. 

## Compile Time Validations

We can validate our infrastructure by ensuring all our compositions and Ludwig libraries compile. That's what the `Makefile` does for you. We'll run through some categories of validation below.

### Compliant Code

`Config/Environment.lw` defines the Environment type, based on the contents of the `ENVIRONMENT` environment variable. Allowed values are `DEV`, `QA`, and `PROD`. To see the compositions compile successfully, set the environment variable and run the Makefile:

```
$ ENVIRONMENT=DEV make
```

You should see no errors or warnings.

![fugue](images/fugue-validate-compilation.gif)

### Environments: Only allow approved environments.

To test the validation for approved environments, set `ENVIRONMENT` to a value that isn't `DEV`, `QA`, or `PROD`, and run the Makefile:

```
$ ENVIRONMENT=DOESNOTEXIST make
```

You should see an error: "ENVIRONMENT must be set to DEV, QA, or PROD."

![fugue](images/fugue-validate-environment.gif)

### Regions: Only allow `us-east-1`.

To test the validation for approved regions, edit `CreateDeveloperEnvironment.lw` and change `AWS.Us-east-1` to `AWS.Eu-west-1` (line 14). Then set the environment variable and run the Makefile:

```
$ ENVIRONMENT=DEV make
```

You should see an error: "Invalid Region: Everything must run in us-east-1."

![fugue](images/fugue-validate-region.gif)

### Ports: SSH allowed for DEV environment *only*, disallowed for QA and PROD.

`InfoSecStandards.lw` sets up `DEV` to be the only approved environment for SSH access. To test this validation, edit `DeveloperEnvironment.lw` and find lines 90-96, shown below:

```
  if InfoSecStandards.isApprovedEnvironment() then
    let sshAccess: [
      EC2.IpPermission.ssh(EC2.IpPermission.Target.securityGroup(clientSg)),
      EC2.IpPermission.ssh(EC2.IpPermission.Target.all),
    ]
    List.concat(rules, sshAccess)
  else rules
```

Replace those lines with these, which remove the if/else statement limiting SSH access to approved environments:

```
  let sshAccess: [
      EC2.IpPermission.ssh(EC2.IpPermission.Target.securityGroup(clientSg)),
      EC2.IpPermission.ssh(EC2.IpPermission.Target.all),
    ]
  List.concat(rules, sshAccess)
```

Make sure the first and last lines are indented two spaces.

Go ahead and set the environment to `DEV` and run the Makefile.
 

```
$ ENVIRONMENT=DEV make
```

It'll compile as normal, because the validation allows SSH access on `DEV`.

Now try setting the environment to `QA` and running the Makefile:

```
$ ENVIRONMENT=QA make
```

You'll see an error: "Inbound SSH access (port 22) is not allowed outside of the DEV environment." This is because even though we tried to allow SSH in other environments by taking away the if/else statement in `DeveloperEnvironment.lw` that limited SSH access to approved environments, the validation in `InfoSecStandards.lw` is still in place.

Try setting the environment to `PROD` and running the Makefile:

```
$ ENVIRONMENT=PROD make
```

Once again, you'll see this error: "Inbound SSH access (port 22) is not allowed outside of the DEV environment."

![fugue](images/fugue-validate-ports.gif)

### Tags: All assets that can be tagged must have specific standard tags applied.

To test the validation for specific standard tags, we'll add a required tag, "PointOfContact", so that our compositions will error unless all resources have that tag. Edit `InfoSecStandards.lw` and add the line `  "PointOfContact"` on line 43, so lines 41-44 look like this:

```
requiredTags: [
  "Name",
  "PointOfContact"
]
```

Then, set the environment to `DEV` and run the Makefile:

```
$ ENVIRONMENT=DEV make
```

You should see an error:

```
  - All taggable resources must have the following tags:
      - Name
      - PointOfContact
```


![fugue](images/fugue-validate-tags.gif)

### InstanceTypes: Limit allowed instances types on a library-by-library basis.

To test the instance type validation, edit `CreateDeveloperEnvironment.lw` and add the line `  instanceType: EC2.M4_16xlarge` on line 31, so that lines 27-32 look like this:

```
devEnvironment: Corp.DeveloperEnvironment.new {
  name: "DevEnvironment",
  keyName: Environment.config.keyName,
  subnets: devNetwork.publicSubnets,
  instanceType: EC2.M4_16xlarge
}
```

Then, set the environment to `DEV` and run the Makefile:

```
$ ENVIRONMENT=DEV make
```

You should see an error: "Failed Compliance Validation: Must be a valid EC2 instance type. Please see InfoSec."

![fugue](images/fugue-validate-instancetype.gif)

### Next Steps

Feel free to explore validations further by browsing `InfoSecStandards.lw`. Experiment with changing those validations, or write new ones. You can also create your own composition incorporating validations.

To learn more about writing Ludwig and using Fugue, [visit our docs site](https://docs.fugue.co/learn.html). If you have any questions, reach out to support@fugue.co.
