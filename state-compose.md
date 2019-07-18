- Feature Name: Function for composing state module functions
- Start Date: 2019-07-17
- SEP Status: Draft
- SEP PR:
- Salt Issue:

# Summary
[summary]: #summary

This is a function for composing salt state module functions from 3 other functions supplied as arguments.

# Motivation
[motivation]: #motivation

The basic idea is to regularize the writing of state module functions as the presently used methods 
tend to be ad hoc and may not separate concerns, such as 
separating the acquisition of data from the question of what to do with it. 

# Design
[design]: #detailed-design

Here is the code of the proposed function (with comments).

```python
def compose_state(get_reference_data, \
                  get_requirements, \
                  operation, \
                  changes_not_required_msg, \
                  changes_required_msg, \
                  retest_fail_msg, \
                  success_msg):
    r'''
    This is a function for composing salt state module functions. The basic idea is to regularize
    the writing of state module functions as the presently used methods tend to
    be ad hoc and may not separate concerns, such as separating the acquisition of data
    from the question of what to do with it. This approach may have limitations and may not
    be suited to every situation.

    The following types can be anything but have to be used consistently:

         - reference data type

         - operation data type

    Args:
         get_reference_data(function):
            
            This function is intended to acquire "reference data" which is supposed to be static data
            pertaining to the desired state. The reference data will be directly used one or two times,
            depending on whether or not an attempt to modify the state is made. In simple cases
            the reference data may just be the name argument. In more complex cases it will be an
            assortment of data derived from the arguments to the function. For instance
            if salt://myfile is the name argument, we may want our function to return the path
            of the cached file as part of the reference data (That way we can use this data more than once
            without re-acquiring it).
             
            Args:
                  name (str):
                     This should be the usual "name" argument passed to state functions
                  **kwargs (keyworkd dict)
                     This is any other data we will need in order to proceed

            Returns:
                  (True, reference data type) or (False, str):
                     This is an ordered pair (A, B). B is either the hoped for
                     reference data or a an error message in the form of a string.

         get_requirements(function):
            
            This function is intended to establish requirements relative to the reference data,
            (e.g. what needs to be installed?). The requirements take the form of a dictionary
            of required changes, and data needed by the operation which will be needed to effect those changes,
            should that dictionary be non-empty. (If the dictionary is empty,
            None will be returned instead of operation data.) This function will be invoked once or twice
            depending on circumstances.
            It will only be ran once if no attempt is made to change the state. It will be ran a second
            time if an attempt were made, so we can see if we really succeeded. (If the operation
            did succeed we confirm that when see see that the requirements returned are just ({}, None):

            Args:

                 reference_data(reference data type):

                     This is the static reference data refered to previously

            Returns:
                 (True, (dict, operation data type or None)) or (False, str):

                     This is an ordered pair (A,B). When A is true, B should be the requirements
                     pair, namely an ordered pair (C,D) such that C is either either the empty
                     dictionary and D is None, or C is a non-empty dictionary and D's type is
                     "operation data type".

         operation (function):

                Args:
                   
                   operation_data (operation data type)
                         
                Returns:

                   (True, None) or (False, str):
                        This is an ordered pair (A,B). When A is True, B is None. 
                        When A is False, B is an error message in the form of a string.
  
         changes_not_required_msg(function):

             Function for generating a message like "All data in reg file 'test_reg' is
             is already in the registry. No action taken."
             
             Args:
           
                reference_data (reference data type)

             Returns:

                str

         changes_required_msg(function):
             
             This function will only be used in test mode when it has been determined that
             changes are required. An example output of this function might be
             "Reg file 'test_reg' needs to be imported." For input the function takes
             the reference data as well as the dictionary of required changes

             Args:

                reference_data (reference data type)

                required_changes (dict)

             Returns:

                str

         retest_fail_msg(function):

             This function will be used when we have performed the operation but the re-test
             to see if we are actually in the desired state has failed. An example output would be
             "Reg file 'test_reg' was imported, but registry is still not in the desired state.".
             For arguments it takes the reference data, as well as a dictionary of changes still
             required.

             Args:
                
                reference_data (reference data type)

                test_required_changes (dict)

             Returns:

                str

         success_msg(function):

             This function will be used when we have attempted to change state and the retest has confirmed
             that the state changed. An example output would be
             "Reg file 'test_reg' has been imported.". For arguments, it takes the reference data, as well
             as a dictionary of changes that have have taken place (this will, in actuality, be the dictionary
             of required changes determined earlier in the process).

             Args:
                
                reference_data (reference data type)

                changes (dict)

             Returns:

                str

         returns(function):

             The return value is a function typed as follows:

             Args:

                 name(str):

                     This should be the usual "name" argument passed to state functions

                  **kwargs(keyword dict)

                     This is any other data we will need in order to gather reference data

             Returns(dict):
          
                 The return value is a standard salt return dictionary.
    '''
    def run_state(name, **kwargs):
        ret = {'name': name,
               'result': False,
               'changes': {},
               'comment': ''}
        (result, reference_data) \
            = get_reference_data(name, **kwargs)
        if not result:
            err_msg = reference_data
            ret['comment'] = err_msg
            ret['result'] = False
            return ret
        # get requirements for operation to proceed
        (result, requirements) \
            = get_requirements(reference_data)
        if not result:
            err_msg = requirements
            ret['comment'] = err_msg
            ret['result'] = False
            return ret
        (required_changes, operation_data) = requirements
        if not required_changes:
            # required_changes is the empty dictionary and we are done
            ret['comment'] = changes_not_required_msg(reference_data)
            ret['result'] = True
            return ret
        if __opts__['test']:
            ret['changes'] = required_changes
            ret['comment'] = changes_required_msg(reference_data, required_changes)
            ret['result'] = None
            return ret
        # Required changes is non-empty and we'ere not in test mode so
        # proceed with the operation
        (result, err_msg) = operation(operation_data)
        if not result:
            # If result is False, that means the operation failed.
            ret['comment'] = err_msg
            ret['result'] = False
            return ret
        # retest
        (result, test_requirements) = get_requirements(reference_data)
        if not result:
            # we met with an error during testing 
            err_msg = test_requirements
            ret['comment'] = err_msg
            ret['result'] = False
            return ret
        (test_required_changes, _) = test_requirements
        if test_required_changes:
            # Test required change dict is *NOT* the empty dictionary even though it should be.
            err_msg = retest_fail_msg(reference_data, test_required_changes)
            ret['comment'] = err_msg
            ret['result'] = False
            return ret
        # Having ran the operation and then tested we can assume that the actual changes
        # now match the required changes we identified earlier.
        ret['changes'] = required_changes
        ret['comment'] = success_msg(reference_data, required_changes)
        ret['result'] = True
        return ret
    return run_state

```

As an example I have taken the windows reg state module and modified it by including
the _compose_state_ function. The three function passed as arguments to _compose_state_ are 
called __imported_reference_data_, __imported_get_requirements_ and 
__imported_do_import_. We end up with a state module function called _imported_ which you can run.
The virtual name of this module is _reg_new_.
You can just drop this in your extension modules directory to get started.

```
# -*- coding: utf-8 -*-
r'''
Manage the Windows registry
===========================

Many python developers think of registry keys as if they were python keys in a
dictionary which is not the case. The windows registry is broken down into the
following components:

Hives
-----

This is the top level of the registry. They all begin with HKEY.

    - HKEY_CLASSES_ROOT (HKCR)
    - HKEY_CURRENT_USER(HKCU)
    - HKEY_LOCAL MACHINE (HKLM)
    - HKEY_USER (HKU)
    - HKEY_CURRENT_CONFIG

Keys
----

Hives contain keys. These are basically the folders beneath the hives. They can
contain any number of subkeys.

When passing the hive\key values they must be quoted correctly depending on the
backslashes being used (``\`` vs ``\\``). The way backslashes are handled in
the state file is different from the way they are handled when working on the
CLI. The following are valid methods of passing the hive\key:

Using single backslashes:
    HKLM\SOFTWARE\Python
    'HKLM\SOFTWARE\Python'

Using double backslashes:
    "HKLM\\SOFTWARE\\Python"

Values or Entries
-----------------

Values or Entries are the name/data pairs beneath the keys and subkeys. All keys
have a default name/data pair. The name is ``(Default)`` with a displayed value
of ``(value not set)``. The actual value is Null.

Example
-------

The following example is taken from the windows startup portion of the registry:

.. code-block:: text

    [HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Run]
    "RTHDVCPL"="\"C:\\Program Files\\Realtek\\Audio\\HDA\\RtkNGUI64.exe\" -s"
    "NvBackend"="\"C:\\Program Files (x86)\\NVIDIA Corporation\\Update Core\\NvBackend.exe\""
    "BTMTrayAgent"="rundll32.exe \"C:\\Program Files (x86)\\Intel\\Bluetooth\\btmshellex.dll\",TrayApp"

In this example these are the values for each:

Hive:
    ``HKEY_LOCAL_MACHINE``

Key and subkeys:
    ``SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Run``

Value:

- There are 3 value names: ``RTHDVCPL``, ``NvBackend``, and ``BTMTrayAgent``
- Each value name has a corresponding value

'''
from __future__ import absolute_import, print_function, unicode_literals

# Import python libs
import logging
import salt.utils.stringutils
import sys
import io
from os.path import basename
from salt.ext.six.moves import configparser
from salt.exceptions import CommandExecutionError
from salt.ext.six import PY2

log = logging.getLogger(__name__)


def __virtual__():
    '''
    Load this state if the reg module exists
    '''
    if 'reg.read_value' not in __utils__:
        return (False, 'reg state module failed to load: '
                       'missing util function: reg.read_value')

    if 'reg.set_value' not in __utils__:
        return (False, 'reg state module failed to load: '
                       'missing util function: reg.set_value')

    if 'reg.delete_value' not in __utils__:
        return (False, 'reg state module failed to load: '
                       'missing util function: reg.delete_value')

    if 'reg.delete_key_recursive' not in __utils__:
        return (False, 'reg state module failed to load: '
                       'missing util function: reg.delete_key_recursive')

    return 'reg_new'


def _parse_key(key):
    '''
    split the hive from the key
    '''
    splt = key.split("\\")
    hive = splt.pop(0)
    key = '\\'.join(splt)
    return hive, key


def present(name,
            vname=None,
            vdata=None,
            vtype='REG_SZ',
            use_32bit_registry=False,
            win_owner=None,
            win_perms=None,
            win_deny_perms=None,
            win_inheritance=True,
            win_perms_reset=False):
    r'''
    Ensure a registry key or value is present.

    Args:

        name (str):
            A string value representing the full path of the key to include the
            HIVE, Key, and all Subkeys. For example:

            ``HKEY_LOCAL_MACHINE\\SOFTWARE\\Salt``

            Valid hive values include:

                - HKEY_CURRENT_USER or HKCU
                - HKEY_LOCAL_MACHINE or HKLM
                - HKEY_USERS or HKU

        vname (str):
            The name of the value you'd like to create beneath the Key. If this
            parameter is not passed it will assume you want to set the
            ``(Default)`` value

        vdata (str, int, list, bytes):
            The value you'd like to set. If a value name (``vname``) is passed,
            this will be the data for that value name. If not, this will be the
            ``(Default)`` value for the key.

            The type of data this parameter expects is determined by the value
            type specified in ``vtype``. The correspondence is as follows:

                - REG_BINARY: Binary data (str in Py2, bytes in Py3)
                - REG_DWORD: int
                - REG_EXPAND_SZ: str
                - REG_MULTI_SZ: list of str
                - REG_QWORD: int
                - REG_SZ: str

                .. note::
                    When setting REG_BINARY, string data will be converted to
                    binary automatically. To pass binary data, use the built-in
                    yaml tag ``!!binary`` to denote the actual binary
                    characters. For example, the following lines will both set
                    the same data in the registry:

                    - ``vdata: Salty Test``
                    - ``vdata: !!binary U2FsdHkgVGVzdA==\n``

                    For more information about the ``!!binary`` tag see
                    `here <http://yaml.org/type/binary.html>`_

            .. note::
                The type for the ``(Default)`` value is always REG_SZ and cannot
                be changed. This parameter is optional. If not passed, the Key
                will be created with no associated item/value pairs.

        vtype (str):
            The value type for the data you wish to store in the registry. Valid
            values are:

                - REG_BINARY
                - REG_DWORD
                - REG_EXPAND_SZ
                - REG_MULTI_SZ
                - REG_QWORD
                - REG_SZ (Default)

        use_32bit_registry (bool):
            Use the 32bit portion of the registry. Applies only to 64bit
            windows. 32bit Windows will ignore this parameter. Default is False.

        win_owner (str):
            The owner of the registry key. If this is not passed, the account
            under which Salt is running will be used.

            .. note::
                Owner is set for the key that contains the value/data pair. You
                cannot set ownership on value/data pairs themselves.

            .. versionadded:: 2019.2.0

        win_perms (dict):
            A dictionary containing permissions to grant and their propagation.
            If not passed the 'Grant` permissions will not be modified.

            .. note::
                Permissions are set for the key that contains the value/data
                pair. You cannot set permissions on value/data pairs themselves.

            For each user specify the account name, with a sub dict for the
            permissions to grant and the 'Applies to' setting. For example:
            ``{'Administrators': {'perms': 'full_control', 'applies_to':
            'this_key_subkeys'}}``. ``perms`` must be specified.

            Registry permissions are specified using the ``perms`` key. You can
            specify a single basic permission or a list of advanced perms. The
            following are valid perms:

                Basic (passed as a string):
                    - full_control
                    - read
                    - write

                Advanced (passed as a list):
                    - delete
                    - query_value
                    - set_value
                    - create_subkey
                    - enum_subkeys
                    - notify
                    - create_link
                    - read_control
                    - write_dac
                    - write_owner

            The 'Applies to' setting is optional. It is specified using the
            ``applies_to`` key. If not specified ``this_key_subkeys`` is used.
            Valid options are:

                Applies to settings:
                    - this_key_only
                    - this_key_subkeys
                    - subkeys_only

            .. versionadded:: 2019.2.0

        win_deny_perms (dict):
            A dictionary containing permissions to deny and their propagation.
            If not passed the `Deny` permissions will not be modified.

            .. note::
                Permissions are set for the key that contains the value/data
                pair. You cannot set permissions on value/data pairs themselves.

            Valid options are the same as those specified in ``win_perms``

            .. note::
                'Deny' permissions always take precedence over 'grant'
                 permissions.

            .. versionadded:: 2019.2.0

        win_inheritance (bool):
            ``True`` to inherit permissions from the parent key. ``False`` to
            disable inheritance. Default is ``True``.

            .. note::
                Inheritance is set for the key that contains the value/data
                pair. You cannot set inheritance on value/data pairs themselves.

            .. versionadded:: 2019.2.0

        win_perms_reset (bool):
            If ``True`` the existing DACL will be cleared and replaced with the
            settings defined in this function. If ``False``, new entries will be
            appended to the existing DACL. Default is ``False``

            .. note::
                Perms are reset for the key that contains the value/data pair.
                You cannot set permissions on value/data pairs themselves.

            .. versionadded:: 2019.2.0

    Returns:
        dict: A dictionary showing the results of the registry operation.

    Example:

    The following example will set the ``(Default)`` value for the
    ``SOFTWARE\\Salt`` key in the ``HKEY_CURRENT_USER`` hive to
    ``2016.3.1``:

    .. code-block:: yaml

        HKEY_CURRENT_USER\\SOFTWARE\\Salt:
          reg.present:
            - vdata: 2016.3.1

    Example:

    The following example will set the value for the ``version`` entry under
    the ``SOFTWARE\\Salt`` key in the ``HKEY_CURRENT_USER`` hive to
    ``2016.3.1``. The value will be reflected in ``Wow6432Node``:

    .. code-block:: yaml

        HKEY_CURRENT_USER\\SOFTWARE\\Salt:
          reg.present:
            - vname: version
            - vdata: 2016.3.1

    In the above example the path is interpreted as follows:

        - ``HKEY_CURRENT_USER`` is the hive
        - ``SOFTWARE\\Salt`` is the key
        - ``vname`` is the value name ('version') that will be created under the key
        - ``vdata`` is the data that will be assigned to 'version'

    Example:

    Binary data can be set in two ways. The following two examples will set
    a binary value of ``Salty Test``

    .. code-block:: yaml

        no_conversion:
          reg.present:
            - name: HKLM\SOFTWARE\SaltTesting
            - vname: test_reg_binary_state
            - vdata: Salty Test
            - vtype: REG_BINARY

        conversion:
          reg.present:
            - name: HKLM\SOFTWARE\SaltTesting
            - vname: test_reg_binary_state_with_tag
            - vdata: !!binary U2FsdHkgVGVzdA==\n
            - vtype: REG_BINARY

    Example:

    To set a ``REG_MULTI_SZ`` value:

    .. code-block:: yaml

        reg_multi_sz:
          reg.present:
            - name: HKLM\SOFTWARE\Salt
            - vname: reg_multi_sz
            - vdata:
              - list item 1
              - list item 2

    Example:

    To ensure a key is present and has permissions:

    .. code-block:: yaml

        set_key_permissions:
          reg.present:
            - name: HKLM\SOFTWARE\Salt
            - vname: version
            - vdata: 2016.3.1
            - win_owner: Administrators
            - win_perms:
                jsnuffy:
                  perms: full_control
                sjones:
                  perms:
                    - read_control
                    - enum_subkeys
                    - query_value
                  applies_to:
                    - this_key_only
            - win_deny_perms:
                bsimpson:
                  perms: full_control
                  applies_to: this_key_subkeys
            - win_inheritance: True
            - win_perms_reset: True
    '''
    ret = {'name': name,
           'result': True,
           'changes': {},
           'comment': ''}

    hive, key = _parse_key(name)

    # Determine what to do
    reg_current = __utils__['reg.read_value'](hive=hive,
                                              key=key,
                                              vname=vname,
                                              use_32bit_registry=use_32bit_registry)

    # Check if the key already exists
    # If so, check perms
    # We check `vdata` and `success` because `vdata` can be None
    if vdata == reg_current['vdata'] and reg_current['success']:
        ret['comment'] = '{0} in {1} is already present' \
                         ''.format(salt.utils.stringutils.to_unicode(vname, 'utf-8') if vname else '(Default)',
                                   salt.utils.stringutils.to_unicode(name, 'utf-8'))
        return __utils__['dacl.check_perms'](
            obj_name='\\'.join([hive, key]),
            obj_type='registry32' if use_32bit_registry else 'registry',
            ret=ret,
            owner=win_owner,
            grant_perms=win_perms,
            deny_perms=win_deny_perms,
            inheritance=win_inheritance,
            reset=win_perms_reset)

    # Cast the vdata according to the vtype
    vdata_decoded = __utils__['reg.cast_vdata'](vdata=vdata, vtype=vtype)

    add_change = {'Key': r'{0}\{1}'.format(hive, key),
                  'Entry': '{0}'.format(salt.utils.stringutils.to_unicode(vname, 'utf-8') if vname else '(Default)'),
                  'Value': vdata_decoded,
                  'Owner': win_owner,
                  'Perms': {'Grant': win_perms,
                            'Deny': win_deny_perms},
                  'Inheritance': win_inheritance}

    # Check for test option
    if __opts__['test']:
        ret['result'] = None
        ret['changes'] = {'reg': {'Will add': add_change}}
        return ret

    # Configure the value
    ret['result'] = __utils__['reg.set_value'](hive=hive,
                                               key=key,
                                               vname=vname,
                                               vdata=vdata,
                                               vtype=vtype,
                                               use_32bit_registry=use_32bit_registry)

    if not ret['result']:
        ret['changes'] = {}
        ret['comment'] = r'Failed to add {0} to {1}\{2}'.format(name, hive, key)
    else:
        ret['changes'] = {'reg': {'Added': add_change}}
        ret['comment'] = r'Added {0} to {1}\{2}'.format(name, hive, key)

    if ret['result']:
        ret = __utils__['dacl.check_perms'](
            obj_name='\\'.join([hive, key]),
            obj_type='registry32' if use_32bit_registry else 'registry',
            ret=ret,
            owner=win_owner,
            grant_perms=win_perms,
            deny_perms=win_deny_perms,
            inheritance=win_inheritance,
            reset=win_perms_reset)

    return ret


def absent(name, vname=None, use_32bit_registry=False):
    r'''
    Ensure a registry value is removed. To remove a key use key_absent.

    Args:
        name (str):
            A string value representing the full path of the key to include the
            HIVE, Key, and all Subkeys. For example:

            ``HKEY_LOCAL_MACHINE\\SOFTWARE\\Salt``

            Valid hive values include:

                - HKEY_CURRENT_USER or HKCU
                - HKEY_LOCAL_MACHINE or HKLM
                - HKEY_USERS or HKU

        vname (str):
            The name of the value you'd like to create beneath the Key. If this
            parameter is not passed it will assume you want to set the
            ``(Default)`` value

        use_32bit_registry (bool):
            Use the 32bit portion of the registry. Applies only to 64bit
            windows. 32bit Windows will ignore this parameter. Default is False.

    Returns:
        dict: A dictionary showing the results of the registry operation.

    CLI Example:

        .. code-block:: yaml

            'HKEY_CURRENT_USER\\SOFTWARE\\Salt':
              reg.absent
                - vname: version

        In the above example the value named ``version`` will be removed from
        the SOFTWARE\\Salt key in the HKEY_CURRENT_USER hive. If ``vname`` was
        not passed, the ``(Default)`` value would be deleted.
    '''
    ret = {'name': name,
           'result': True,
           'changes': {},
           'comment': ''}

    hive, key = _parse_key(name)

    # Determine what to do
    reg_check = __utils__['reg.read_value'](hive=hive,
                                            key=key,
                                            vname=vname,
                                            use_32bit_registry=use_32bit_registry)
    if not reg_check['success'] or reg_check['vdata'] == '(value not set)':
        ret['comment'] = '{0} is already absent'.format(name)
        return ret

    remove_change = {'Key': r'{0}\{1}'.format(hive, key),
                     'Entry': '{0}'.format(vname if vname else '(Default)')}

    # Check for test option
    if __opts__['test']:
        ret['result'] = None
        ret['changes'] = {'reg': {'Will remove': remove_change}}
        return ret

    # Delete the value
    ret['result'] = __utils__['reg.delete_value'](hive=hive,
                                                  key=key,
                                                  vname=vname,
                                                  use_32bit_registry=use_32bit_registry)
    if not ret['result']:
        ret['changes'] = {}
        ret['comment'] = r'Failed to remove {0} from {1}'.format(key, hive)
    else:
        ret['changes'] = {'reg': {'Removed': remove_change}}
        ret['comment'] = r'Removed {0} from {1}'.format(key, hive)

    return ret


def key_absent(name, use_32bit_registry=False):
    r'''
    .. versionadded:: 2015.5.4

    Ensure a registry key is removed. This will remove the key, subkeys, and all
    value entries.

    Args:

        name (str):
            A string representing the full path to the key to be removed to
            include the hive and the keypath. The hive can be any of the
            following:

                - HKEY_LOCAL_MACHINE or HKLM
                - HKEY_CURRENT_USER or HKCU
                - HKEY_USER or HKU

        use_32bit_registry (bool):
            Use the 32bit portion of the registry. Applies only to 64bit
            windows. 32bit Windows will ignore this parameter. Default is False.

    Returns:
        dict: A dictionary showing the results of the registry operation.


    CLI Example:

        The following example will delete the ``SOFTWARE\DeleteMe`` key in the
        ``HKEY_LOCAL_MACHINE`` hive including all its subkeys and value pairs.

        .. code-block:: yaml

            remove_key_demo:
              reg.key_absent:
                - name: HKEY_CURRENT_USER\SOFTWARE\DeleteMe

        In the above example the path is interpreted as follows:

            - ``HKEY_CURRENT_USER`` is the hive
            - ``SOFTWARE\DeleteMe`` is the key
    '''
    ret = {'name': name,
           'result': True,
           'changes': {},
           'comment': ''}

    hive, key = _parse_key(name)

    # Determine what to do
    if not __utils__['reg.read_value'](hive=hive,
                                       key=key,
                                       use_32bit_registry=use_32bit_registry)['success']:
        ret['comment'] = '{0} is already absent'.format(name)
        return ret

    ret['changes'] = {
        'reg': {
            'Removed': {
                'Key': r'{0}\{1}'.format(hive, key)}}}

    # Check for test option
    if __opts__['test']:
        ret['result'] = None
        return ret

    # Delete the value
    __utils__['reg.delete_key_recursive'](hive=hive,
                                          key=key,
                                          use_32bit_registry=use_32bit_registry)
    if __utils__['reg.read_value'](hive=hive,
                                   key=key,
                                   use_32bit_registry=use_32bit_registry)['success']:
        ret['result'] = False
        ret['changes'] = {}
        ret['comment'] = 'Failed to remove registry key {0}'.format(name)

    return ret


def _imported_parse_reg_file(reg_file):
    r'''
    This is a utility function used by imported_file. This parses a reg file and returns a
    ConfigParser object. A configparser.Error exception will be thrown implicitly
    upon failure to parse.
    '''
    reg_file_fp = io.open(reg_file, "r", encoding="utf-16")
    try:
        reg_file_fp.readline()
        # The first line of a reg file is english text which we must consume before parsing.
        # It contains no data.
        reg_data = configparser.ConfigParser()
        if PY2:
            reg_data.readfp(reg_file_fp)
        else:
            reg_data.read_file(reg_file_fp)
        return reg_data
    finally:
        reg_file_fp.close()


def _imported_reference_data_wrk(reference_reg_file_url, use_32bit_registry):
    r'''
    This is a utility function used by imported. It does the real work
    for the function _imported_reference_data. It caches the file pointed to by the
    file pointed to with the reference_reg_file_url argument and the parses it.
    It then returns (reference_reg_file, reference_parse, use_32bit_registry)
    where reference_parse is the parser object.
    This information is considered to be the "reference data". It is acquired once
    (here) and then used throughout the process.
    '''
    reference_reg_file = __salt__['cp.cache_file'](reference_reg_file_url)
    if not reference_reg_file:
        error_str = "File/URL '{0}' probably invalid.".format(reference_reg_file_url)
        raise ValueError(error_str)
    reference_parse = _imported_parse_reg_file(reference_reg_file)
    reference_section_count = len(reference_parse.sections())
    if reference_section_count == 0:
        error_str_fmt = "File/URL '{0}' has a section count of 0. It may not be a valid REG file."
        error_str = error_str_fmt.format(reference_reg_file_url)
        raise ValueError(error_str)
    return (reference_reg_file, reference_parse, use_32bit_registry)


def _imported_compose_cmd_execution_err_msg(err):
    r'''
    This is a utility function used by imported. It composes an error message
    based on a CommandExecutionError exception. It is expected that
    the info dictionary of err will have a "command" entry.
    '''
    comment_fmt = "{0}. The attempted command was '{1}'."
    comment = comment_fmt.format(err.message, err.info.get("command", "(Unknown)"))
    return comment


def _imported_reference_data(reference_reg_file_url, **kwargs):
    r'''
    This is a utility function used by imported. It essentially wraps
    imported_reference_data_wrk and it handles some possible exceptions.
    The return value of this function is an ordered pair (A,B).
    The values of A and B are defined as follows:
       A: True if the data is successfully acquired, False otherwise
       B: The reference data, if it has been acquired, otherwise an error message
    '''
    use_32bit_registry = kwargs["use_32bit_registry"]
    try:
        reference_data \
            = _imported_reference_data_wrk(reference_reg_file_url, use_32bit_registry)
    except ValueError as err:
        comment = str(err)
        return (False, comment)
    except CommandExecutionError as err:
        comment = _imported_compose_cmd_execution_err_msg(err)
        return (False, comment)
    except configparser.Error:
        comment_fmt = "Could not parse file/URL '{0}'. It may not be a valid REG file."
        comment = comment_fmt.format(reference_reg_file_url)
        return (False, comment)
    return (True, reference_data)


def _imported_state_parse(reg_location, use_32bit_registry):
    r'''
    This is a utility function used by imported. This function takes a registry
    location as its primary argument. Using that it exports from the registry location
    (using reg export) to a temporary file and it then parses the file. If reg export
    fails, it raises a CommandExecution error exception. If the function succeeds,
    it returns a parser object representing the registry at the location.
    '''
    if use_32bit_registry:
        word_sz_txt = "32"
    else:
        word_sz_txt = "64"
    present_reg_file = (__salt__['temp.file'])()
    try:
        cmd = 'reg export "{0}" "{1}" /y /reg:{2}'.format(reg_location, present_reg_file, word_sz_txt)
        cmd_ret_dict = __salt__['cmd.run_all'](cmd, python_shell=True)
        retcode = cmd_ret_dict['retcode']
        if retcode != 0:
            cmd_ret_dict['command'] = cmd
            raise CommandExecutionError(
                'reg.exe export failed from registry location {0}'.format(reg_location),
                info=cmd_ret_dict)
        present_data = _imported_parse_reg_file(present_reg_file)
        return present_data
    finally:
        (__salt__['file.remove'])(present_reg_file)

    
def _imported_state_data(reg_location, use_32bit_registry):
    r'''
    This is a utility function used by imported. Its primary argument is a registry location.
    It first tests to see if the location exists in the registry. If it does not,
    it returns None. If the location does exist, this returns a parser object representing
    the registry at the location (by invoking _imported_state_parse).
    '''
    reg_hive = reg_location[:reg_location.index("\\")]
    reg_key_path = reg_location[reg_location.index("\\")+1:]
    if __salt__['reg.key_exists'](reg_hive, reg_key_path, use_32bit_registry):
        state_data = _imported_state_parse(reg_location, use_32bit_registry)
    else:
        state_data = None
    return state_data    


def _imported_get_value_changes(reference_parse, state_parse, key_path, \
                                new_values, old_values):
    r'''
    This is a utility function used by imported.
    Determines which "values" (name/data pairs) have been changed or added,
    under a specific key path.
    '''
    reference_items = reference_parse.items(key_path)
    for (reference_option, reference_value) in reference_items:
        if not state_parse or not state_parse.has_option(key_path, reference_option):
            if key_path not in new_values:
                new_values[key_path] = []
            new_values[key_path].append({reference_option.strip('"'): reference_value.strip('"')})
        else:
            state_value = state_parse.get(key_path, reference_option)
            if reference_value != state_value:
                if key_path not in old_values:
                    old_values[key_path] = []
                old_values[key_path].append({reference_option.strip('"'): state_value.strip('"')})
                if key_path not in new_values:
                    new_values[key_path] = []
                new_values[key_path].append({reference_option.strip('"'): reference_value.strip('"')})
    return True


def _imported_get_changes(state_parse, reference_parse):
    r'''
    This is a utility function used by imported.
    Its job is to construct the changes dictionary. It does this by comparing
    parse trees. One is of data we exported from the registry and the other
    is a parse tree of our reg file.
    '''
    new_keys = []
    new_values = {}
    old_values = {}
    for key_path in reference_parse.sections():
        if not state_parse or not state_parse.has_section(key_path):
            # state_parse could be None if the location doesn't exist in the registry
            new_keys.append(key_path)
        _imported_get_value_changes(reference_parse, state_parse, key_path, \
                                    new_values, old_values)
    result = {}
    if new_keys or new_values or old_values:
    # old_values is actually redundant in the condition
        result["old"] = {}
        result["old"]["values"] = old_values
        result["new"] = {}
        result["new"]["added keys"] = new_keys
        result["new"]["added/changed values"] = new_values
    return result


def _imported_do_import(operation_data):
    r'''
    This is a utility function used by imported. This function calls
    the module function reg.import_file. If that function then throws
    exceptions, this function will catch some of them.
    The function returns an ordered pair (A,B).
    The values are defined as follows
       A: Whether or not the operation succeeded expressed as a boolean.
       B: an error message if the operation failed. Otherwise None.
    '''
    (reference_reg_file, use_32bit_registry) = operation_data
    try:
        __salt__['reg.import_file'](reference_reg_file, use_32bit_registry)
    except ValueError as err:
        err_msg_fmt = "Call to module function 'reg.import_file' has failed. Error is '{0}'"
        err_msg = comment_fmt.format(str(err))
        return (False, err_msg)
    except CommandExecutionError as err:
        err_msg_fmt = "Call to module function 'reg.import_file' has failed. Error is '{0}'."
        err_msg = comment_fmt.format(err.message)
        return (False, err_msg)
    return (True, None)
    

def _imported_get_requirements(reference_data):
    r'''
    This is a utility function used by imported. It's job is to
    determine, using the the reference data, what
    changes are required, and if changes are required, the data needed by
    the operation. Specifically an ordered pair A, B is returned. A and B are defined
    as follows.
           A: If an error occurs the value of A will be False, otherwise True
           B: If an error occurs the value will be an error message,
              otherwise the value will be an ordered pair (c,d). The values
              of c and d are defined as follows.
                 c: This is a dictionary of required changes. It is not a dictionary
                    of changes that have happened.
                 d: If c is non-empty, d is data required by the operation. In this
                    case that data is just the path of the cached reg file.
                    (needed by reg.import).
    '''
    (reference_reg_file, reference_parse, use_32bit_registry) \
        = reference_data
    reg_location = (reference_parse.sections())[0]
    try:
        state_parse = _imported_state_data(reg_location, use_32bit_registry)
    except CommandExecutionError as err:
        comment = _imported_compose_cmd_execution_err_msg(err)
        return (False, comment)
    changes = _imported_get_changes(state_parse,reference_parse)
    operation_data = None
    if changes:
        operation_data = (reference_reg_file, use_32bit_registry)
    return (True, (changes, operation_data))


def compose_state(get_reference_data, \
                  get_requirements, \
                  operation, \
                  changes_not_required_msg, \
                  changes_required_msg, \
                  retest_fail_msg, \
                  success_msg):
    r'''
    This is a function for composing salt state module functions. The basic idea is to regularize
    the writing of state module functions as the presently used methods tend to
    be ad hoc and may not separate concerns, such as separating the acquisition of data
    from the question of what to do with it. This approach may have limitations and may not
    be suited to every situation.

    The following types can be anything but have to be used consistently:

         - reference data type

         - operation data type

    Args:
         get_reference_data(function):
            
            This function is intended to acquire "reference data" which is supposed to be static data
            pertaining to the desired state. The reference data will be directly used one or two times,
            depending on whether or not an attempt to modify the state is made. In simple cases
            the reference data may just be the name argument. In more complex cases it will be an
            assortment of data dredged from the system.
             
            Args:
                  name (str):
                     This should be the usual "name" argument passed to state functions
                  **kwargs (keyworkd dict)
                     This is any other data we will need in order to proceed

            Returns:
                  (True, reference data type) or (False, str):
                     This is an ordered pair (A, B). B is either the hoped for
                     reference data or a an error message in the form of a string.

         get_requirements(function):
            
            This function is intended to establish requirements, namely a dictionary
            of required changes, and data needed by the operation which will be needed to effect those changes,
            should that dictionary be non-empty. (If the dictionary is empty,
            None will be returned instead of operation data.) This function will be invoked once or twice
            depending on circumstances.
            It will only be ran once if no attempt is made to change the state. It will be ran a second
            time if an attempt were made, so we can see if we really succeeded. (If the operation
            did succeed we confirm that when see see that the requirements returned are just ({}, None):

            Args:

                 reference_data(reference data type):

                     This is the static reference data refered to previously

            Returns:
                 (True, (dict, operation data type or None)) or (False, str):

                     This is an ordered pair (A,B). When A is true, B should be the requirements
                     pair, namely an ordered pair (C,D) such that C is either either the empty
                     dictionary and D is None, or C is a non-empty dictionary and D's type is
                     "operation data type".

         operation (function):

                Args:
                   
                   operation_data (operation data type)
                         
                Returns:

                   (True, None) or (False, str):
                        This is an ordered pair (A,B). When A is True, B is None. 
                        When A is False, B is an error message in the form of a string.
  
         changes_not_required_msg(function):

             Function for generating a message like "All data in reg file 'test_reg' is
             is already in the registry. No action taken."
             
             Args:
           
                reference_data (reference data type)

             Returns:

                str

         changes_required_msg(function):
             
             This function will only be used in test mode when it has been determined that
             changes are required. An example output of this function might be
             "Reg file 'test_reg' needs to be imported." For input the function takes
             the reference data as well as the dictionary of required changes

             Args:

                reference_data (reference data type)

                required_changes (dict)

             Returns:

                str

         retest_fail_msg(function):

             This function will be used when we have performed the operation but the re-test
             to see if we are actually in the desired state has failed. An example output would be
             "Reg file 'test_reg' was imported, but registry is still not in the desired state.".
             For arguments it takes the reference data, as well as a dictionary of changes still
             required.

             Args:
                
                reference_data (reference data type)

                test_required_changes (dict)

             Returns:

                str

         success_msg(function):

             This function will be used when we have attempted to change state and the retest has confirmed
             that the state changed. An example output would be
             "Reg file 'test_reg' has been imported.". For arguments, it takes the reference data, as well
             as a dictionary of changes that have have taken place (this will, in actuality, be the dictionary
             of required changes determined earlier in the process).

             Args:
                
                reference_data (reference data type)

                changes (dict)

             Returns:

                str

         returns(function):

             The return value is a function typed as follows:

             Args:

                 name(str):

                     This should be the usual "name" argument passed to state functions

                  **kwargs(keyword dict)

                     This is any other data we will need in order to gather reference data

             Returns(dict):
          
                 The return value is a standard salt return dictionary.
    '''
    def run_state(name, **kwargs):
        ret = {'name': name,
               'result': False,
               'changes': {},
               'comment': ''}
        (result, reference_data) \
            = get_reference_data(name, **kwargs)
        if not result:
            err_msg = reference_data
            ret['comment'] = err_msg
            ret['result'] = False
            return ret
        # get requirements for operation to proceed
        (result, requirements) \
            = get_requirements(reference_data)
        if not result:
            err_msg = requirements
            ret['comment'] = err_msg
            ret['result'] = False
            return ret
        (required_changes, operation_data) = requirements
        if not required_changes:
            # required_changes is the empty dictionary and we are done
            ret['comment'] = changes_not_required_msg(reference_data)
            ret['result'] = True
            return ret
        if __opts__['test']:
            ret['changes'] = required_changes
            ret['comment'] = changes_required_msg(reference_data, required_changes)
            ret['result'] = None
            return ret
        # Required changes is non-empty and we'ere not in test mode so
        # proceed with the operation
        (result, err_msg) = operation(operation_data)
        if not result:
            # If result is False, that means the operation failed.
            ret['comment'] = err_msg
            ret['result'] = False
            return ret
        # retest
        (result, test_requirements) = get_requirements(reference_data)
        if not result:
            # we met with an error during testing 
            err_msg = test_requirements
            ret['comment'] = err_msg
            ret['result'] = False
            return ret
        (test_required_changes, _) = test_requirements
        if test_required_changes:
            # Test required change dict is *NOT* the empty dictionary even though it should be.
            err_msg = retest_fail_msg(reference_data, test_required_changes)
            ret['comment'] = err_msg
            ret['result'] = False
            return ret
        # Having ran the operation and then tested we can assume that the actual changes
        # now match the required changes we identified earlier.
        ret['changes'] = required_changes
        ret['comment'] = success_msg(reference_data, required_changes)
        ret['result'] = True
        return ret
    return run_state


def imported(name, use_32bit_registry=False):
    r'''
    .. versionadded:: Neon

    This is intended to be the stateful correlate of ``reg.import_file``. This will import
    a ``REG`` file (by invoking ``reg.import_file``) only if the import will create changes
    to the registry.

    Args:

        name (str):
            The full path of the ``REG`` file. This can be either a local file
            path or a URL type supported by salt (e.g. ``salt://salt_master_path``)
        use_32bit_registry (bool):
            If the value of this parameter is ``True``, and if the import
            proceeds, the ``REG`` file
            will be imported into the Windows 32 bit registry. Otherwise the
            Windows 64 bit registry will be used.

    ..note::
        This function is reliant on a conventionally created reg file (either
        an export of a key from ``regedit``, or the result of ``reg export <key>``.
        A reg file with multiple tree roots will not work with this function.
    '''
    ret_f = compose_state(_imported_reference_data, \
                          _imported_get_requirements, \
                          _imported_do_import, \
                          
                          lambda reference_data : \
                          "All data in reg file '{0}' is already in the registry. No action taken." \
                          .format(basename(reference_data[0])), \

                          lambda reference_data, required_changes : \
                          "Reg file '{0}' needs to be imported." \
                          .format(basename(reference_data[0])), \

                          lambda reference_data, test_required_changes: \
                          "Import of '{0}' appears to have succeeded, but registry is still not in the desired state." \
                          .format(basename(reference_data[0])), \

                          lambda reference_data, changes : \
                          "Reg file '{0}' has been imported." \
                          .format(basename(reference_data[0])))
    
    return ret_f(name, **{"use_32bit_registry": use_32bit_registry})


```

You can use the following windows reg file to test with (it will need to be in WIndows UTF16 encoding to work).
You can, in reality, use any reg file you want, so long as its a conventional reg file with a single root.

```
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\SaltTestKey2]
@="default_val"
"A test value name"="a test value data"
"A test value name88"="xmore dataxyzugodwhystop"
"c"="dxhlep"
"x"="yif"
"e"="c"

[HKEY_LOCAL_MACHINE\SOFTWARE\SaltTestKey2\subKey]


[HKEY_LOCAL_MACHINE\SOFTWARE\SaltTestKey2\subKey3]
@="default_val"
"A test value name"="a test value data"
"A test value name88"="xmore dataxyzugodwhystop"
"c"="dxhlep"
"x"="yuf"
"e"="c"


[HKEY_LOCAL_MACHINE\SOFTWARE\SaltTestKey2\subKey4]
@="default_val"
"A test value name"="a test value data"
"A test value name88"="xmore dataxyzugodwhystop"
"c"="dxhlep"
"x"="yuf"
"e"="c"
```

Next you can create a state file such as:

```
test state module:
   reg_new.imported:
       - name: salt://win/reg_test_new.reg

```

You can then see it working by running
```
/usr/local/bin/salt windows_minion state.apply test
```

## Alternatives
[alternatives]: #alternatives

I do not know what alternatives have been considered. The impact of not doing this is the status quo
where a multiplicity of different workflows are employed to create state functions. I think that this
has lead to bugs and hard to understand code.

## Unresolved questions
[unresolved]: #unresolved-questions

I have resolved all of my own questions.

# Drawbacks
[drawbacks]: #drawbacks

Certainly in many cases there could perhaps be slightly more code when this feature is used,
just because we have to define these functions and then decode reference data and operation data
into more meaningful bits of data.
Another drawback of this design is that you will never get a populated changes dictionary _and_
an error at the same time. This is consistent with what happens in functional languages where we have
an an either Monad (i.e. You either get something or an error message) so I don't think of this
as particularly bad. But if people think this is a dealbreaker I'll consider whatever changes
have to be made. Note that the non-error message results are generated from the reference data
and a change dictionary. Some may think of this as a drawback but if the change dictionaries really have
good data, then it should be possible to generate a good message.
Ultimately there will be no real costs unless people choose to use this feature and then regret it later.
There will be no requirement for this feature to be used by anyone.
