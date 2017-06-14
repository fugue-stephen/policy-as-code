# Policy As Code

## Compile Time Validations

We can validate our infrastructure by ensuring all our compositions and Ludwig libraries compile. That's what the `Makefile` does for you. We'll run through some categories of validation below. All validations are defined in `InfoSecStandards.lw`.

### Compliant Code

`Environment.lw` defines the Environment type, based on the contents of the `ENVIRONMENT` environment variable. Allowed values are `DEV`, `QA`, and `PROD`. To see the compositions compile successfully, set the environment variable and run the Makefile:

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
