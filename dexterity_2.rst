Dexterity Types II: Growing up
==============================

.. sidebar:: Get the code!

    Get the code for this chapter (:doc:`More info <sneak>`) using this command in the buildout-directory:

    .. code-block:: bash

        cp -Rf src/ploneconf.site_sneak/chapters/23_dexterity_2/ src/ploneconf.site


The existing talks are still lacking some functionality we want to use.

In this part we will:

* add a marker-interface to our talk-type
* create custom catalog-indexes
* query the catalog for them
* enable some more default-features for our type


Add a marker-interface to the talk-type
---------------------------------------

Marker Interfaces
+++++++++++++++++

The content-type `Talk` is not yet a first-class citizen because it does not implement his own interface. Interfaces are like name-tags, telling other elements who and what you are and what you can do. A marker interface is like such a nametag. The talks actually have a auto-generated marker-interface ``plone.dexterity.schema.generated.Plone_0_talk``.

The problem is that the name of the Plone-instance ``Plone`` is part of that interface-name. If you now moved these types to a site with another name, code that uses these Interfaces would no longer find the objects in question.

To solve this we add a new file ``interfaces.py``:

.. code-block:: python
    :linenos:

    from zope.interface import Interface


    class ITalk(Interface):
        """Marker interface for Talks
        """

``ITalk`` is a marker-interface. We can bind Views and Viewlets to content that provide these interfaces. Lets see how we can provide this Interface. There are two solution for this.

1. Let them be instances of a class that implements this Interface.
2. Register this interface as a behavior and enable it on talks.

The first option has a important drawback: Only new talks would be instances of the new class. We would either have to migrate the existing talks or delete them.

So lets register the interface as a behavior in ``behaviors/configure.zcml``

.. code-block:: xml

  <plone:behavior
    title="Talk"
    description="Marker interface for talks to be able to bind views to."
    provides="..interfaces.ITalk"
    />

And enable it on the type in ``profiles/default/types/talk.xml``

.. code-block:: xml
    :linenos:
    :emphasize-lines: 5

    <property name="behaviors">
     <element value="plone.app.dexterity.behaviors.metadata.IDublinCore"/>
     <element value="plone.app.content.interfaces.INameFromTitle"/>
     <element value="ploneconf.site.behaviors.social.ISocial"/>
     <element value="ploneconf.site.interfaces.ITalk"/>
    </property>

Either reinstall the addon, apply the behavior by hand or run a upgrade-step (see below) and the interface will be there.

Then we can safely bind the talkview to the new marker interface.

.. code-block:: xml
    :emphasize-lines: 3

    <browser:page
      name="talkview"
      for="ploneconf.site.interfaces.ITalk"
      layer="zope.interface.Interface"
      class=".views.TalkView"
      template="templates/talkview.pt"
      permission="zope2.View"
      />

Now the ``/talkview`` can only be used on objects that implement said interface. We can now also query the catalog for objects providing this interface ``catalog(object_provides="ploneconf.site.interfaces.ITalk")``. The `talklistview` and the `demoview` do not get this constraint since they are not only used on talks.

.. note::

    Just for completeness sake, this is what would have to happen for the first option:

    * Create a new class that inherits from ``plone.dexterity.content.Container`` and implements the marker interface.

      .. code-block:: python

          from plone.dexterity.content import Container
          from ploneconf.site.interfaces import ITalk
          from zope.interface import implements

          class Talk(Container):
              implements(ITalk)

    * Modify the class for new talks in ``profiles/default/types/talk.xml``

      .. code-block:: xml
          :linenos:
          :emphasize-lines: 3

          ...
          <property name="add_permission">cmf.AddPortalContent</property>
          <property name="klass">ploneconf.site.content.talk.Talk</property>
          <property name="behaviors">
          ...

    * Create a upgrade step to modify the class of existing types. A code-example on how to do this is in `ftw.upgrade <https://github.com/4teamwork/ftw.upgrade/blob/master/ftw/upgrade/step.py#L270>`_.

Upgrade-steps
-------------

When projects evolve you'll sometimes have to modify various things while the site is already up and brimming with content and users. Upgrade steps are pieces of code that run when upgrading from one version of a addon to a newer one. They can do just about anything.

We will create a upgrade step that

* runs the typeinfo-step (i.e. loads the GenericSetup configuration stores in ``profiles/default/types.xml`` and ``profiles/default/types/...`` so we don't have to reinstall the addon to have our changes from above take effect) and
* cleans up some content that might be scattered around the site in the early stages of creating it. We will move all talks to a folder ``talks`` (unless they already are there).

Upgrade steps are usually registered in their own zcml-file. Create ``upgrades.zcml``

.. code-block:: xml
    :linenos:

    <configure
      xmlns="http://namespaces.zope.org/zope"
      xmlns:i18n="http://namespaces.zope.org/i18n"
      xmlns:genericsetup="http://namespaces.zope.org/genericsetup"
      i18n_domain="ploneconf.site">

      <genericsetup:upgradeStep
        title="Update and cleanup talks"
        description="Updates typeinfo and moves talks to a folder 'talks'"
        source="1"
        destination="1001"
        handler="ploneconf.site.upgrades.upgrade_site"
        sortkey="1"
        profile="ploneconf.site:default"
        />

    </configure>

The upgrade step bumps the verion-number of the GenericSetup profile of ploneconf.site from 1 to 1001. The version is stored in ``profiles/default/metadata.xml``. Change it to

..  code-block:: xml

    <version>1001</version>

Include the new ``upgrades.zcml`` in our ``configure.zcml`` by adding:

..  code-block:: xml

    <include file="upgrades.zcml" />

GenericSetup now expects the code as a method ``upgrade_site`` in the file ``upgrades.py``. Let's create it.

..  code-block:: python
    :linenos:

    from plone import api
    import logging

    default_profile = 'profile-ploneconf.site:default'
    logger = logging.getLogger('ploneconf.site')


    def upgrade_site(setup):
        setup.runImportStepFromProfile(default_profile, 'typeinfo')
        catalog = api.portal.get_tool('portal_catalog')
        portal = api.portal.get()
        if 'the-event' not in portal:
            theevent = api.content.create(
                container=portal,
                type='Folder',
                id='the-event',
                title='The event')
        else:
            theevent = portal['the-event']
        if 'talks' not in theevent:
            talks = api.content.create(
                container=theevent,
                type='Folder',
                id='talks',
                title='Talks')
        else:
            talks = theevent['talks']
        talks_url = talks.absolute_url()
        brains = catalog(portal_type='talk')
        for brain in brains:
            if talks_url in brain.getURL():
                continue
            obj = brain.getObject()
            logger.info('Moving %s to %s' % (obj.absolute_url(), talks.absolute_url()))
            api.content.move(
                source=obj,
                target=talks,
                safe_id=True)

After restarting the site we can run the step:

* Go to the addon-controlpanel http://localhost:8080/Plone/prefs_install_products_form. There should now be a warning **This add-on has been upgraded. Old profile version was 1. New profile version is 1001** and a button next to it.
* Run the upgrade-step by clicking on it.

On the console you should see logging-messages like::

    INFO ploneconf.site Moving http://localhost:8080/Plone/old-talk1 to http://localhost:8080/Plone/the-event/talks

Alternatively you can select which upgrade-steps to run like this:

* In the ZMI got to *portal_setup*
* Go to the tab *Upgrades*
* Select *ploneconf.site* from the dropdown and click *Choose profile*
* Run the upgrade step.

.. seealso::

    http://docs.plone.org/develop/addons/components/genericsetup.html#id1


.. note::

    Upgrading from an older version of Plone to a newer one also runs upgrade steps from the package ``plone.app.upgrade``. You should be able to upgrade a clean site from 2.5 to 5.0a2 with a click.

    For an example see the upgrade-step to Plone 5.0a1 https://github.com/plone/plone.app.upgrade/blob/master/plone/app/upgrade/v50/alphas.py#L23



Add a browserlayer
------------------

A browserlayer is another such marker-interface. Browserlayers allow us to easily enable and disable views and other site functionality based on installed add-ons and themes.

Since we want the features we write only to be available when ploneconf.site actually is installed we can bind them to a browserlayer.

In ``interfaces.py`` we add:

.. code-block:: python

    class IPloneconfSiteLayer(Interface):
        """Marker interface for the Browserlayer
        """

We register the browserlayer in GenericSetup in ``profiles/default/browserlayer.xml``

.. code-block:: xml

    <?xml version="1.0"?>
    <layers>
      <layer name="ploneconf.site"
        interface="ploneconf.site.interfaces.IPloneconfSiteLayer" />
    </layers>

After reinstalling the addon we can bind the talkview, the demoview and the talklistview to our layer. Here is an example using the talkview.

.. code-block:: xml
    :emphasize-lines: 4

    <browser:page
      name="talklistview"
      for="ploneconf.site.interfaces.ITalk"
      layer="..interfaces.IPloneconfSiteLayer"
      class=".views.TalkListView"
      template="templates/talklistview.pt"
      permission="zope2.View"
      />

Note the relative python-path ``..interfaces.IPloneconfSiteLayer``. It is equivalent to the absolute path ``ploneconf.site.interfaces.IPloneconfSiteLayer``.

.. seealso::

    http://docs.plone.org/develop/plone/views/layers.html


Exercise
++++++++

Do you need to bind the social-viewlet from chapter 20 to this new browser-layer?

..  admonition:: Solution
    :class: toggle

    No, it would make no difference since the viewlet is already bound to the marker interface ``ploneconf.site.behaviors.social.ISocial``.

Add catalog-indexes
-------------------

In the `talklistview` we had to wake up all objects to access some of their attributes. That is ok if we don't have many objects and they are light dexterity-objects. If we had thousands of objects this might not be a good idea.

Instead of loading them all into memory we will use catalog-indexes to get the data we want to display.

Add a new file ``profiles/default/catalog.xml``

.. code-block:: xml
    :linenos:

    <?xml version="1.0"?>
    <object name="portal_catalog">
      <index name="type_of_talk" meta_type="FieldIndex">
        <indexed_attr value="type_of_talk"/>
      </index>
      <index name="speaker" meta_type="FieldIndex">
        <indexed_attr value="speaker"/>
      </index>
      <index name="audience" meta_type="KeywordIndex">
        <indexed_attr value="audience"/>
      </index>

      <column value="audience" />
      <column value="type_of_talk" />
      <column value="speaker" />
    </object>

This adds new indexes for the three fields we want to show in the listing. Not that *audience* is a ``KeywordIndex`` because the field is multi-valued, but we want a separate index-entry for every value in on an object.

The ``column ..`` entry allows us to display these values of these indexes in the tableview of collections.

.. note::

    Until Plone 4.3.2 adding indexes in catalog.xml was harmful because reinstalling the addon purged the indexes! See http://www.starzel.de/blog/a-reminder-about-catalog-indexes.

    To run additional custom code on (re-)installing an addon you should use a `setuphandler.py <http://docs.plone.org/develop/addons/components/genericsetup.html#custom-installer-code-setuphandlers-py>`_.

* Reinstall the addon
* Go to http://localhost:8080/Plone/portal_catalog/manage_catalogIndexes to inspect populate and inspect the new indexes

.. seealso::

    http://docs.plone.org/develop/plone/searching_and_indexing/indexing.html


Query for custom indexes
------------------------

The new indexes behave like the ones that plone has built in:

.. code-block:: python

    >>> (Pdb) from Products.CMFCore.utils import getToolByName
    >>> (Pdb) catalog = getToolByName(self.context, 'portal_catalog')
    >>> (Pdb) catalog(type_of_talk='Keynote')
    [<Products.ZCatalog.Catalog.mybrains object at 0x10737b9a8>, <Products.ZCatalog.Catalog.mybrains object at 0x10737b9a8>]
    >>> (Pdb) catalog(audience=('Advanced', 'Professionals'))
    [<Products.ZCatalog.Catalog.mybrains object at 0x10737b870>, <Products.ZCatalog.Catalog.mybrains object at 0x10737b940>, <Products.ZCatalog.Catalog.mybrains object at 0x10737b9a8>]
    >>> (Pdb) brain = catalog(type_of_talk='Keynote')[0]
    >>> (Pdb) brain.speaker
    u'David Glick'

We now can use the new indexes to improve the talklistview so we don't have to wake up the objects any more.

.. code-block:: python
    :linenos:

    class TalkListView(BrowserView):
        """ A list of talks
        """

        def talks(self):
            results = []
            portal_catalog = getToolByName(self.context, 'portal_catalog')
            current_path = "/".join(self.context.getPhysicalPath())

            brains = portal_catalog(portal_type="talk",
                                    path=current_path)
            for brain in brains:
                results.append({
                    'title': brain.Title,
                    'description': brain.Description,
                    'url': brain.getURL(),
                    'audience': ', '.join(brain.audience or []),
                    'type_of_talk': brain.type_of_talk,
                    'speaker': brain.speaker,
                    'uuid': brain.UID,
                    })
            return results

The template does not need to be changed and the result did not change as well.

Add collection criteria
-----------------------

To be able to search content in collections using the new indexes we would have to register them as criteria for the querystring-widget that collections use.

Add a new file ``profiles/default/registry.xml``

.. code-block:: xml
    :linenos:

    <registry>
      <records interface="plone.app.querystring.interfaces.IQueryField"
               prefix="plone.app.querystring.field.audience">
        <value key="title">Audience</value>
        <value key="description">A custom speaker index</value>
        <value key="enabled">True</value>
        <value key="sortable">False</value>
        <value key="operations">
          <element>plone.app.querystring.operation.string.is</element>
        </value>
        <value key="group">Metadata</value>
      </records>
      <records interface="plone.app.querystring.interfaces.IQueryField"
               prefix="plone.app.querystring.field.type_of_talk">
        <value key="title">Type of Talk</value>
        <value key="description">A custom index</value>
        <value key="enabled">True</value>
        <value key="sortable">False</value>
        <value key="operations">
          <element>plone.app.querystring.operation.string.is</element>
        </value>
        <value key="group">Metadata</value>
      </records>
    </registry>

.. seealso::

  http://docs.plone.org/develop/plone/functionality/collections.html#add-new-collection-criteria-new-style-plone-app-collection-installed


Add more features through generic-setup
---------------------------------------

Enable versioning and a diff-view for talks through GenericSetup.

Add new file ``profiles/default/repositorytool.xml``

.. code-block:: xml
    :linenos:

    <?xml version="1.0"?>
    <repositorytool>
      <policymap>
        <type name="talk">
          <policy name="at_edit_autoversion"/>
          <policy name="version_on_revert"/>
        </type>
      </policymap>
    </repositorytool>


Add new file ``profiles/default/diff_tool.xml``

.. code-block:: xml
    :linenos:

    <?xml version="1.0"?>
    <object>
      <difftypes>
        <type portal_type="talk">
          <field name="any" difftype="Compound Diff for Dexterity types"/>
        </type>
      </difftypes>
    </object>
