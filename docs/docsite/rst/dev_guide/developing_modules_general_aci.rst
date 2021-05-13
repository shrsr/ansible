.. _aci_dev_guide:

****************************
Developing Cisco ACI modules
****************************
This is a thorough walk-through of how to create new Cisco ACI modules for Ansible.

For more information about Cisco ACI, look at the :ref:`Cisco ACI user guide <aci_guide>`.

What's covered in this section:

.. contents::
   :depth: 3
   :local:


.. _aci_dev_guide_intro:

Introduction
============
The `cisco.aci collection <https://galaxy.ansible.com/cisco/aci>`_ already includes a large number of Cisco ACI modules, however the ACI object model is huge and covering all possible functionality would easily cover more than 1500 individual modules. Therefore, Cisco develops modules asked by people on a just in time basis.

If you need specific functionality, you have 3 options:

- Open an issue using https://github.com/CiscoDevNet/ansible-aci/issues/new/choose so that Cisco developers can build, enhance or fix the modules for you
- Learn the ACI object model and use the low-level APIC REST API using the :ref:`aci_rest <aci_rest_module>` module
- Contribute to the Cisco DevNet's ansible-aci by writing your own dedicated modules and be a part of the Cisco ansible-aci community

.. _aci_dev_guide_git:

Let’s look at how we can retrieve the current version of collection. 

Fork, Clone and Branch
======================

A fork is a copy of a repository that allows you to make changes to the repository without affecting the original project.
You can contribute to the original project using Pull Requests from the forked repository.

* Go to: https://github.com/CiscoDevNet/ansible-aci
* Fork CiscoDevnet’s **ansible-aci** repo. 

.. seealso::

   `_How to fork a repo: <https://docs.github.com/en/github/getting-started-with-github/fork-a-repo>`_
   
  
Clone allows you to copy a repository to your local machine. 

* Clone the forked repo by going to terminal and enter: 
.. code-block:: Blocks

   git clone https://github.com/<Forked Repo>/ansible-aci.git


* Add the main repository "upstream"

"origin" is name for your forked repo, from which you push and pull while "upstream" is a name for the main repo, from where you pull and keep a clone of your fork updated, but you don't have push access to it. Adding the main repository "upstream" is a one time operation.
.. code-block:: Blocks

   git remote add upstream https://github.com/CiscoDevNet/aci-go-client.git


Creating branches makes it easier to fix bugs, add new features and integrate new versions after they have been tested in isolation. Master is the default
branch of the local repository. Each time you need to make changes to a module we need to create a new branch from master.

* Create a branch from master by using the following commands on the terminal and add the main repo **upstream**:
.. code-block:: Blocks
   
   git master
   git checkout -b <branch-name> 
   git remote add upstream https://github.com/CiscoDevNet/ansible-aci.git

* Go to **ansible-aci -> plugins -> modules** folder. The new module goes in this folder.

The modules folder consists of modules that cover a specific functionality of the objects in ACI. The module_utils folder has the aci.py file which serves as a library for the modules. Most modules in the collection borrow functions from this library.

So let's look at how a typical ACI module is built.

.. _aci_dev_guide_module_structure:

ACI module structure
====================

Importing objects from Python libraries
---------------------------------------
The following imports are standard across ACI modules:

.. code-block:: python

    from ansible.module_utils.aci import ACIModule, aci_argument_spec
    from ansible.module_utils.basic import AnsibleModule


Defining the argument spec
--------------------------
The first line adds the standard connection parameters to the module. After that, the next section will update the ``argument_spec`` dictionary with module-specific parameters. The module-specific parameters should include:

* the object_id (usually the name)
* the configurable properties of the object
* the parent object IDs (all parents up to the root)
* only child classes that are a 1-to-1 relationship (1-to-many/many-to-many require their own module to properly manage)
* the state

  + ``state: absent`` to ensure object does not exist
  + ``state: present`` to ensure the object and configs exist; this is also the default
  + ``state: query`` to retrieve information about objects in the class

.. code-block:: python

    def main():
        argument_spec = aci_argument_spec()
        argument_spec.update(
            object_id=dict(type='str', aliases=['name']),
            object_prop1=dict(type='str'),
            object_prop2=dict(type='str', choices=['choice1', 'choice2', 'choice3']),
            object_prop3=dict(type='int'),
            parent_id=dict(type='str'),
            child_object_id=dict(type='str'),
            child_object_prop=dict(type='str'),
            state=dict(type='str', default='present', choices=['absent', 'present', 'query']),
        )


.. hint:: Do not provide default values for configuration arguments. Default values could cause unintended changes to the object.

Using the AnsibleModule object
------------------------------
The following section creates an AnsibleModule instance. The module should support check-mode, so we pass the ``argument_spec`` and  ``supports_check_mode`` arguments. Since these modules support querying the APIC for all objects of the module's class, the object/parent IDs should only be required if ``state: absent`` or ``state: present``.

.. code-block:: python

    module = AnsibleModule(
        argument_spec=argument_spec,
        supports_check_mode=True,
        required_if=[
            ['state', 'absent', ['object_id', 'parent_id']],
            ['state', 'present', ['object_id', 'parent_id']],
        ],
    )


Mapping variable definition
---------------------------
Once the AnsibleModule object has been initiated, the necessary parameter values should be extracted from ``params`` and any data validation should be done. Usually the only params that need to be extracted are those related to the ACI object configuration and its child configuration. If you have integer objects that you would like to validate, then the validation should be done here, and the ``ACIModule.payload()`` method will handle the string conversion.

.. code-block:: python

    object_id = object_id
    object_prop1 = module.params['object_prop1']
    object_prop2 = module.params['object_prop2']
    object_prop3 = module.params['object_prop3']
    if object_prop3 is not None and object_prop3 not in range(x, y):
        module.fail_json(msg='Valid object_prop3 values are between x and (y-1)')
    child_object_id = module.params[' child_objec_id']
    child_object_prop = module.params['child_object_prop']
    state = module.params['state']


Using the ACIModule object
--------------------------
The ACIModule class handles most of the logic for the ACI modules. The ACIModule extends functionality to the AnsibleModule object, so the module instance must be passed into the class instantiation.

.. code-block:: python

    aci = ACIModule(module)

The ACIModule has six main methods that are used by the modules:

* construct_url
* get_existing
* payload
* get_diff
* post_config
* delete_config

The first two methods are used regardless of what value is passed to the ``state`` parameter.

Constructing URLs
^^^^^^^^^^^^^^^^^
The ``construct_url()`` method is used to dynamically build the appropriate URL to interact with the object, and the appropriate filter string that should be appended to the URL to filter the results.

* When the ``state`` is not ``query``, the URL is the base URL to access the APIC plus the distinguished name to access the object. The filter string will restrict the returned data to just the configuration data.
* When ``state`` is ``query``, the URL and filter string used depends on what parameters are passed to the object. This method handles the complexity so that it is easier to add new modules and so that all modules are consistent in what type of data is returned.

.. note:: Our design goal is to take all ID parameters that have values, and return the most specific data possible. If you do not supply any ID parameters to the task, then all objects of the class will be returned. If your task does consist of ID parameters sed, then the data for the specific object is returned. If a partial set of ID parameters are passed, then the module will use the IDs that are passed to build the URL and filter strings appropriately.

The ``construct_url()`` method takes 2 required arguments:

* **self** - passed automatically with the class instance
* **root_class** - A dictionary consisting of ``aci_class``, ``aci_rn``, ``target_filter``, and ``module_object`` keys

  + **aci_class**: The name of the class used by the APIC, for example ``fvTenant``

  + **aci_rn**: The relative name of the object, for example ``tn-ACME``

  + **target_filter**: A dictionary with key-value pairs that make up the query string for selecting a subset of entries, for example ``{'name': 'ACME'}``

  + **module_object**: The particular object for this class, for example ``ACME``

Example:

.. code-block:: python

    aci.construct_url(
        root_class=dict(
            aci_class='fvTenant',
            aci_rn='tn-{0}'.format(tenant),
            target_filter={'name': tenant},
            module_object=tenant,
        ),
    )

Some modules, like ``aci_tenant``, are the root class and so they would not need to pass any additional arguments to the method.

The ``construct_url()`` method takes 4 optional arguments, the first three imitate the root class as described above, but are for child objects:

* subclass_1 - A dictionary consisting of ``aci_class``, ``aci_rn``, ``target_filter``, and ``module_object`` keys

  + Example: Application Profile Class (AP)

* subclass_2 - A dictionary consisting of ``aci_class``, ``aci_rn``, ``target_filter``, and ``module_object`` keys

  + Example: End Point Group (EPG)

* subclass_3 - A dictionary consisting of ``aci_class``, ``aci_rn``, ``target_filter``, and ``module_object`` keys

  + Example: Binding a Contract to an EPG

* child_classes - The list of APIC names for the child classes supported by the modules.

  + This is a list, even if it is a list of one
  + These are the unfriendly names used by the APIC
  + These are used to limit the returned child_classes when possible
  + Example: ``child_classes=['fvRsBDSubnetToProfile', 'fvRsNdPfxPol']``

.. note:: Sometimes the APIC will require special characters ([, ], and -) or will use object metadata in the name ("vlanns" for VLAN pools); the module should handle adding special characters or joining of multiple parameters in order to keep expected inputs simple.

Getting the existing configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Once the URL and filter string have been built, the module is ready to retrieve the existing configuration for the object:

* ``state: present`` retrieves the configuration to use as a comparison against what was entered in the task. All values that are different than the existing values will be updated.
* ``state: absent`` uses the existing configuration to see if the item exists and needs to be deleted.
* ``state: query`` uses this to perform the query for the task and report back the existing data.

.. code-block:: python

    aci.get_existing()


When state is present
^^^^^^^^^^^^^^^^^^^^^
When ``state: present``, the module needs to perform a diff against the existing configuration and the task entries. If any value needs to be updated, then the module will make a POST request with only the items that need to be updated. Some modules have children that are in a 1-to-1 relationship with another object; for these cases, the module can be used to manage the child objects.

Building the ACI payload
""""""""""""""""""""""""
The ``aci.payload()`` method is used to build a dictionary of the proposed object configuration. All parameters that were not provided a value in the task will be removed from the dictionary (both for the object and its children). Any parameter that does have a value will be converted to a string and added to the final dictionary object that will be used for comparison against the existing configuration.

The ``aci.payload()`` method takes two required arguments and 1 optional argument, depending on if the module manages child objects.

* ``aci_class`` is the APIC name for the object's class, for example ``aci_class='fvBD'``
* ``class_config`` is the appropriate dictionary to be used as the payload for the POST request

  + The keys should match the names used by the APIC.
  + The values should be the corresponding value in ``module.params``; these are the variables defined above

* ``child_configs`` is optional, and is a list of child config dictionaries.

  + The child configs include the full child object dictionary, not just the attributes configuration portion.
  + The configuration portion is built the same way as the object.

.. code-block:: python

    aci.payload(
        aci_class=aci_class,
        class_config=dict(
            name=bd,
            descr=description,
            type=bd_type,
        ),
        child_configs=[
            dict(
                fvRsCtx=dict(
                    attributes=dict(
                        tnFvCtxName=vrf
                    ),
                ),
            ),
        ],
    )


Performing the request
""""""""""""""""""""""
The ``get_diff()`` method is used to perform the diff, and takes only one required argument, ``aci_class``.
Example: ``aci.get_diff(aci_class='fvBD')``

The ``post_config()`` method is used to make the POST request to the APIC if needed. This method doesn't take any arguments and handles check mode.
Example: ``aci.post_config()``


Example code
""""""""""""
.. code-block:: text

    if state == 'present':
        aci.payload(
            aci_class='<object APIC class>',
            class_config=dict(
                name=object_id,
                prop1=object_prop1,
                prop2=object_prop2,
                prop3=object_prop3,
            ),
            child_configs=[
                dict(
                    '<child APIC class>'=dict(
                        attributes=dict(
                            child_key=child_object_id,
                            child_prop=child_object_prop
                        ),
                    ),
                ),
            ],
        )

        aci.get_diff(aci_class='<object APIC class>')

        aci.post_config()


When state is absent
^^^^^^^^^^^^^^^^^^^^
If the task sets the state to absent, then the ``delete_config()`` method is all that is needed. This method does not take any arguments, and handles check mode.

.. code-block:: text

        elif state == 'absent':
            aci.delete_config()


Exiting the module
^^^^^^^^^^^^^^^^^^
To have the module exit, call the ACIModule method ``exit_json()``. This method automatically takes care of returning the common return values for you.

.. code-block:: text

        aci.exit_json()

    if __name__ == '__main__':
        main()

Documentation Section
---------------------
All the parameters defined in the argument_spec like the object_id, configurable properties of the object, parent object IDs, state etc. need to be documented in the same file as the module. The format of documentation is shown below:

.. code-block:: yaml

   DOCUMENTATION = r'''
   ---
   module: aci_<name_of_module>
   short_description: Short description of the module being created (config:<name_of_class>)
   description:
   - Functionality one
   - Functionality two
   options:
     object_id:
       description:
       - Description of object
       type: data type of object eg. 'str'
       aliases: [ Alternate name of the object ]
     object_prop1:
       description:
       - Description of property one
       type: Property's data type eg. 'int'
       choices: [ choice one, choice two ]
     object_prop2:
       description:
       - Description of property two
       type: Property's data type eg. 'bool'
     state:
       description:
       - Use C(present) or C(absent) for adding or removing.
       - Use C(query) for listing an object or multiple objects.
       type: str
       choices: [ absent, present, query ]
       default: present
   extends_documentation_fragment:
   - cisco.aci.aci

Examples Section
----------------
Examples section must consist of Ansible tasks which can be used as a reference to build playbooks. The format of this section is shown below:

.. code-block:: yaml

   EXAMPLES = r'''
   - name: Add a new object
     cisco.aci.aci_<name_of_module>:
       host: apic
       username: admin
       password: SomeSecretePassword
       object_id: id
       object_prop1: prop1
       object_prop2: prop2
       state: present
      delegate_to: localhost

   - name: Remove an object
     cisco.aci.aci_<name_of_module>:
       host: apic
       username: admin
       password: SomeSecretePassword
       object_id: id
       object_prop1: prop1
       object_prop2: prop2
       state: absent
      delegate_to: localhost

   - name: Query an object
     cisco.aci.aci_<name_of_module>:
       host: apic
       username: admin
       password: SomeSecretePassword
       object_id: id
       state: query
      delegate_to: localhost

   - name: Query all objects
     cisco.aci.aci_<name_of_module>:
       host: apic
       username: admin
       password: SomeSecretePassword
       state: query
      delegate_to: localhost
   '''

Example Module
==============

The following example consists of Documentation, Examples and Module Sections discussed above. All these sections must be present in a single file: **aci_<aci-module-name>.py** which goes inside the **modules** folder.

.. code-block:: python

      #!/usr/bin/python
      # -*- coding: utf-8 -*-

      # Copyright: (c) <year>, <Name> (@<github id>)
      # GNU General Public License v3.0+ (see LICENSE or https://www.gnu.org/licenses/gpl-3.0.txt)

      from __future__ import absolute_import, division, print_function
      __metaclass__ = type

      ANSIBLE_METADATA = {'metadata_version': '1.1',
                          'status': ['preview'],
                          'supported_by': 'community'}

      DOCUMENTATION = r'''
      ---
      module: aci_l2out
      short_description: Manage Layer2 Out (L2Out) objects.
      description:
      - Manage Layer2 Out configuration on Cisco ACI fabrics.
      options:
        tenant:
          description:
          - Name of an existing tenant.
          type: str
        l2out:
          description:
          - The name of outer layer2.
          type: str
          aliases: [ 'name' ]
        description:
          description:
          - Description for the L2Out.
          type: str
        bd:
          description:
          - Name of the Bridge domain which is associted with the L2Out.
          type: str
        domain:
          description:
          - Name of the external L2 Domain that is being associated with L2Out.
          type: str
        vlan:
          description:
          - The VLAN which is being associated with the L2Out.
          type: int
        state:
          description:
          - Use C(present) or C(absent) for adding or removing.
          - Use C(query) for listing an object or multiple objects.
          type: str
          choices: [ absent, present, query ]
          default: present
        name_alias:
          description:
          - The alias for the current object. This relates to the nameAlias field in ACI.
          type: str
      extends_documentation_fragment:
      - cisco.aci.aci

      notes:
      - The C(tenant) must exist before using this module in your playbook.
        The M(cisco.aci.aci_tenant) modules can be used for this.
      seealso:
      - name: APIC Management Information Model reference
        description: More information about the internal APIC class B(fvTenant).
        link: https://developer.cisco.com/docs/apic-mim-ref/
      author:
      - <Author's Name> (@<github id>)
      '''

      EXAMPLES = r'''
      - name: Add a new L2Out
        cisco.aci.aci_l2out:
          host: apic
          username: admin
          password: SomeSecretePassword
          tenant: Auto-Demo
          l2out: l2out
          description: via Ansible
          bd: bd1
          domain: l2Dom
          vlan: 3200
          state: present
          delegate_to: localhost

      - name: Remove an L2Out
        cisco.aci.aci_l2out:
          host: apic
          username: admin
          password: SomeSecretePassword
          tenant: Auto-Demo
          l2out: l2out
          state: absent
          delegate_to: localhost

      - name: Query an L2Out
        cisco.aci.aci_l2out:
          host: apic
          username: admin
          password: SomeSecretePassword
          tenant: Auto-Demo
          l2out: l2out
          state: query
          delegate_to: localhost
          register: query_result

      - name: Query all L2Outs in a specific tenant
        cisco.aci.aci_l2out:
          host: apic
          username: admin
          password: SomeSecretePassword
          tenant: Auto-Demo
          state: query
          delegate_to: localhost
          register: query_result
      '''

      RETURN = r'''
         current:
           description: The existing configuration from the APIC after the module has finished
           returned: success
           type: list
           sample:
             [
                 {
                     "fvTenant": {
                         "attributes": {
                             "descr": "Production environment",
                             "dn": "uni/tn-production",
                             "name": "production",
                             "nameAlias": "",
                             "ownerKey": "",
                             "ownerTag": ""
                         }
                     }
                 }
             ]
         error:
           description: The error information as returned from the APIC
           returned: failure
           type: dict
           sample:
             {
                 "code": "122",
                 "text": "unknown managed object class foo"
             }
         raw:
           description: The raw output returned by the APIC REST API (xml or json)
           returned: parse error
           type: str
           sample: '<?xml version="1.0" encoding="UTF-8"?><imdata totalCount="1"><error code="122" text="unknown managed object class "/></imdata>'
         sent:
           description: The actual/minimal configuration pushed to the APIC
           returned: info
           type: list
           sample:
             {
                 "fvTenant": {
                     "attributes": {
                         "descr": "Production environment"
                     }
                 }
             }
         previous:
           description: The original configuration from the APIC before the module has started
           returned: info
           type: list
           sample:
             [
                 {
                     "fvTenant": {
                         "attributes": {
                             "descr": "Production",
                             "dn": "uni/tn-production",
                             "name": "production",
                             "nameAlias": "",
                             "ownerKey": "",
                             "ownerTag": ""
                         }
                     }
                 }
             ]
         proposed:
           description: The assembled configuration from the user-provided parameters
           returned: info
           type: dict
           sample:
             {
                 "fvTenant": {
                     "attributes": {
                         "descr": "Production environment",
                         "name": "production"
                     }
                 }
             }
         filter_string:
           description: The filter string used for the request
           returned: failure or debug
           type: str
           sample: ?rsp-prop-include=config-only
         method:
           description: The HTTP method used for the request to the APIC
           returned: failure or debug
           type: str
           sample: POST
         response:
           description: The HTTP response from the APIC
           returned: failure or debug
           type: str
           sample: OK (30 bytes)
         status:
           description: The HTTP status from the APIC
           returned: failure or debug
           type: int
           sample: 200
         url:
           description: The HTTP url used for the request to the APIC
           returned: failure or debug
           type: str
           sample: https://10.11.12.13/api/mo/uni/tn-production.json
         '''

      from ansible.module_utils.basic import AnsibleModule
      from ansible_collections.cisco.aci.plugins.module_utils.aci import ACIModule, aci_argument_spec


      def main():
          argument_spec = aci_argument_spec()
          argument_spec.update(
              bd=dict(type='str'),
              l2out=dict(type='str', aliases=['name']),
              domain=dict(type='str'),
              vlan=dict(type='int'),
              description=dict(type='str'),
              state=dict(type='str', default='present', choices=['absent', 'present', 'query']),
              tenant=dict(type='str'),
              name_alias=dict(type='str'),
          )

          module = AnsibleModule(
              argument_spec=argument_spec,
              supports_check_mode=True,
              required_if=[
                  ['state', 'absent', ['l2out', 'tenant']],
                  ['state', 'present', ['bd', 'l2out', 'tenant', 'domain', 'vlan']],
              ],
          )

          bd = module.params.get('bd')
          l2out = module.params.get('l2out')
          description = module.params.get('description')
          domain = module.params.get('domain')
          vlan = module.params.get('vlan')
          state = module.params.get('state')
          tenant = module.params.get('tenant')
          name_alias = module.params.get('name_alias')
          child_classes = ['l2extRsEBd', 'l2extRsL2DomAtt', 'l2extLNodeP']

          aci = ACIModule(module)
          aci.construct_url(
              root_class=dict(
                  aci_class='fvTenant',
                  aci_rn='tn-{0}'.format(tenant),
                  module_object=tenant,
                  target_filter={'name': tenant},
              ),
              subclass_1=dict(
                  aci_class='l2extOut',
                  aci_rn='l2out-{0}'.format(l2out),
                  module_object=l2out,
                  target_filter={'name': l2out},
              ),
              child_classes=child_classes,
          )

          aci.get_existing()

          if state == 'present':
              child_configs = [
                  dict(
                      l2extRsL2DomAtt=dict(
                          attributes=dict(
                              tDn='uni/l2dom-{0}'.format(domain)
                          )
                      )
                  ),
                  dict(
                      l2extRsEBd=dict(
                          attributes=dict(
                              tnFvBDName=bd, encap='vlan-{0}'.format(vlan)
                          )
                      )
                  )
              ]

              aci.payload(
                  aci_class='l2extOut',
                  class_config=dict(
                      name=l2out,
                      descr=description,
                      dn='uni/tn-{0}/l2out-{1}'.format(tenant, l2out),
                      nameAlias=name_alias
                  ),
                  child_configs=child_configs,
              )

              aci.get_diff(aci_class='l2extOut')

              aci.post_config()

          elif state == 'absent':
              aci.delete_config()

          aci.exit_json()


      if __name__ == "__main__":
          main()


.. _aci_dev_guide_testing:

Testing ACI library functions
=============================
You can test your ``construct_url()`` and ``payload()`` arguments without accessing APIC hardware by using the following python script:

.. code-block:: text

    #!/usr/bin/python
    import json
    from ansible.module_utils.network.aci.aci import ACIModule

    # Just another class mimicing a bare AnsibleModule class for construct_url() and payload() methods
    class AltModule():
        params = dict(
            host='dummy',
            port=123,
            protocol='https',
            state='present',
            output_level='debug',
        )

    # A sub-class of ACIModule to overload __init__ (we don't need to log into APIC)
    class AltACIModule(ACIModule):
        def __init__(self):
            self.result = dict(changed=False)
            self.module = AltModule()
            self.params = self.module.params

    # Instantiate our version of the ACI module
    aci = AltACIModule()

    # Define the variables you need below
    aep = 'AEP'
    aep_domain = 'uni/phys-DOMAIN'

    # Below test the construct_url() arguments to see if it produced correct results
    aci.construct_url(
        root_class=dict(
            aci_class='infraAttEntityP',
            aci_rn='infra/attentp-{}'.format(aep),
            target_filter={'name': aep},
            module_object=aep,
        ),
        subclass_1=dict(
            aci_class='infraRsDomP',
            aci_rn='rsdomP-[{}]'.format(aep_domain),
            target_filter={'tDn': aep_domain},
            module_object=aep_domain,
        ),
    )

    # Below test the payload arguments to see if it produced correct results
    aci.payload(
        aci_class='infraRsDomP',
        class_config=dict(tDn=aep_domain),
    )

    # Print the URL and proposed payload
    print 'URL:', json.dumps(aci.url, indent=4)
    print 'PAYLOAD:', json.dumps(aci.proposed, indent=4)


This will result in:

.. code-block:: yaml

    URL: "https://dummy/api/mo/uni/infra/attentp-AEP/rsdomP-[phys-DOMAIN].json"
    PAYLOAD: {
        "infraRsDomP": {
            "attributes": {
                "tDn": "phys-DOMAIN"
            }
        }
    }
    
Creating a test file for the ACI module
=======================================
* Go to **ansible-aci -> tests -> intergartion -> targets**
* Create a folder having the same name as the module of the format: `aci_<aci-module-name>`
* Create a folder: **tasks**, inside `aci_<aci-module-name>`
* Create a yml file: **main.yml** inside **tasks**
* The **main.yml** will serve as the test file for the new module. It should have all the tasks that comprises all functions in the new module. 

Example is provided below for reference.

The following test file verifies the Layer2 Out configuration on ACI module:

.. code-block:: yaml

   # Test code for the ACI modules
   # Copyright: (c) <year>, <Name> (@<github id>)

   # GNU General Public License v3.0+ (see LICENSE or https://www.gnu.org/licenses/gpl-3.0.txt)

   - name: Test that we have an ACI APIC host, ACI username and ACI password
     fail:
       msg: 'Please define the following variables: aci_hostname, aci_username and aci_password.'
     when: aci_hostname is not defined or aci_username is not defined or aci_password is not defined

   # GET Credentials from the inventory
   - name: Set vars
      set_fact: 
      aci_info: &aci_info
       host: "{{ aci_hostname }}"
       username: "{{ aci_username }}"
       password: "{{ aci_password }}"
       validate_certs: '{{ aci_validate_certs | default(false) }}'
       use_ssl: '{{ aci_use_ssl | default(true) }}'
       use_proxy: '{{ aci_use_proxy | default(true) }}'
       output_level: debug

   # CLEAN ENVIRONMENT
   - name: Remove ansible_tenant if it already exists
     aci_tenant:
       <<: *aci_info 
       tenant: ansible_tenant
       state: absent

   - name: Add a new tenant required for l2out
      aci_tenant:
       <<: *aci_info 
       tenant: ansible_tenant
       description: Ansible tenant
       state: present

   # ADD l2out 
   - name: Add L2Out
     aci_l2out:
       <<: *aci_info
       tenant: ansible_tenant
       l2out: ansible_l2out
       description: Test deployment 
       bd: ansible_bd
       domain: l2Dom
       vlan: 3200
       state: present
     register: add_l2out

   - name: Verify that ansible_l2out has been created with correct attributes
     assert:
       that:
       - add_l2out.current.0.l2extOut.attributes.dn == "uni/tn-ansible_tenant/l2out-ansible_l2out"
       - add_l2out.current.0.l2extOut.attributes.name == "ansible_l2out"

   # ADD l2out again to check idempotency
   - name: Add the L2Out again
     aci_l2out:
       <<: *aci_info
       tenant: ansible_tenant
       l2out: ansible_l2out
       description: Test deployment 
       bd: ansible_bd
       domain: l2Dom
       vlan: 3200
       state: present
     register: add_l2out_again

   - name: Verify that ansible_l2out stays the same
     assert:
       that:
       - add_l2out_again is not changed

   # QUERY l2out
   - name: Query the L2Out  
     aci_l2out:
       <<: *aci_info
       tenant: ansible_tenant
       l2out: ansible_l2out
       state: query
     register: query_l2out

   - name: Verify the attributes under query_l2out
     assert:
       that:
       - query_l2out is not changed
       - query_l2out.current.0.l2extOut.attributes.dn == "uni/tn-ansible_tenant/l2out-ansible_l2out"
       - query_l2out.current.0.l2extOut.attributes.name == "ansible_l2out"

   - name: Query all l2outs under a specific tenant
     aci_l2out:
       <<: *aci_info
       tenant: ansible_tenant
       state: query
     register: query_l2out_all

   - name: Verify query_l2out_all
     assert:
       that:
       - query_l2out_all is not changed

   # DELETE l2out
   - name: Remove the L2Out 
     aci_l2out:
       <<: *aci_info
       tenant: ansible_tenant
       l2out: ansible_l2out
       state: absent
     register: remove_l2out

   - name: Verify remove_l2out
     assert:
       that:
       - remove_l2out is changed
       - remove_l2out.previous.0.l2extOut.attributes.dn == "uni/tn-ansible_tenant/l2out-ansible_l2out"
       - remove_l2out.previous.0.l2extOut.attributes.name == "ansible_l2out"
       
Sanity checks, Testing ACI Integration and Generating Coverage Report
---------------------------------------------------------------------

* Go to **ansible-aci -> tests -> intergartion -> inventory.networking** and update the file

.. code-block:: ini

   [aci]
   <apic-label-name> ansible_host=<apic-host> ansible_connection=local aci_hostname=<apic-host> 
   aci_username=<apic-username> aci_password= <apic-password>

* Go to **ansible-aci** on terminal and test the new module using the following commands:

.. code-block:: Blocks

      ansible-galaxy collection build --force
      ansible-galaxy collection install cisco-aci-* --force
      cd ~/.ansible/collections/ansible_collections/cisco/aci
      ansible-test sanity --docker --color --truncate 0 -v --coverage
      ansible-test network-integration --docker --color --truncate 0 -vvv --coverage aci_<your module name>
      ansible-test coverage report
      ansible-test coverage html
      open ~/.ansible/collections/ansible_collections/cisco/aci/tests/output/reports/coverage/index.html

* Commit and Push the code to your forked repo:
The following git commands are for reference:

.. code-block:: Blocks
    
       git status
       git add <new-files>
       git commit -m <commit message>
       git fetch upstream master
       git rebase upstream/master
       git push origin <branch-name>


* Make a pull request from your forked repo to the original repo.

.. seealso::

   `ACI Fundamentals: ACI Policy Model <https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/1-x/aci-fundamentals/b_ACI-Fundamentals/b_ACI-Fundamentals_chapter_010001.html>`_
       A good introduction to the ACI object model.
   `APIC Management Information Model reference <https://developer.cisco.com/docs/apic-mim-ref/>`_
       Complete reference of the APIC object model.
   `APIC REST API Configuration Guide <https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/2-x/rest_cfg/2_1_x/b_Cisco_APIC_REST_API_Configuration_Guide.html>`_
       Detailed guide on how the APIC REST API is designed and used, incl. many examples.

