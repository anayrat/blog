---
# Leave the homepage title empty to use the site title
title: ''
date: 2022-10-24
type: landing

sections:
  - block: hero
    demo: true # Only display this section in the Wowchemy demo site
    content:
      title: DBA PostgreSQL Freelance
      #image:
      #  filename: hero-academic.png
      cta:
        label: Plus d'informations
        url: "#about"
      cta_alt:
        label:
        url: https://discord.gg/z8wNYzb
      cta_note:
        label:
      text: Vous chercher un DBA PosgreSQL Freelance ?
    design:
      background:
        gradient_end: '#1976d2'
        gradient_start: '#004ba0'
        text_color_light: true
  - block: about.biography
    id: about
    content:
      title: Présentation
      # Choose a user profile to display (a folder name within `content/authors/`)
      username: adrien
  - block: skills
    content:
      title: Compétences
      text: ''
      # Choose a user to display skills from (a folder name within `content/authors/`)
      username: adrien
    design:
      columns: '1'
  - block: experience
    content:
      title: Experiences
      # Date format for experience
      #   Refer to https://wowchemy.com/docs/customization/#date-format
      date_format: Jan 2006
      # Experiences.
      #   Add/remove as many `experience` items below as you like.
      #   Required fields are `title`, `company`, and `date_start`.
      #   Leave `date_end` empty if it's your current employer.
      #   Begin multi-line descriptions with YAML's `|2-` multi-line prefix.
      items:
        - title: DBA PostgreSQL Freelance
          company: Freelance
          company_url: ''
          company_logo:
          location: Valence (Remote)
          date_start: '2021-01-01'
          date_end: ''
          description: |2-
              Diverses missions courtes d'expertises (Audit, résolution d'incident, Remote DBA, Optimisation...)

              Compétences:

              * PostgreSQL
              * SQL
              * Optimisation des requêtes
              * Database Reliability Engineering
              * Amazon RDS
              * Google Cloud Platform (GCP)
              * Change Data Capture (CDC)
              * Bases de données
              * Performance
              * Benchmark
              * pgBackRest
              * Production
              * Datadog
              * Modélisation

        - title: DBA PostgreSQL Freelance
          company: Europ Assistance  (Freelance)
          company_url: ''
          company_logo:
          location: Valence (Remote)
          date_start: '2022-10-01'
          date_end: ''
          description: |2-
              Audit et expertise sur PostgreSQL (RDS) : optimisation, conseils, réduction des coût sur AWS, montée de version avec downtime réduit.

              Compétences:

              * PostgreSQL
              * SQL
              * Optimisation des requêtes
              * Database Reliability Engineering
              * Amazon RDS
              * Change Data Capture (CDC)
              * Bases de données
              * Performance
              * Benchmark
              * Production
              * Datadog
              * Modélisation

        - title: DBA PostgreSQL Freelance
          company: Alma (Freelance)
          company_url: ''
          company_logo:
          location: Valence (Remote)
          date_start: '2022-05-01'
          date_end: ''
          description: |2-
              Accompagnement d'Alma sur l'utilisation de PostgreSQL (cloudSQL): revue système monitoring (Datadog) optimisation de requêtes, conseils schéma

              Compétences:

              * PostgreSQL
              * SQL
              * Optimisation des requêtes
              * Database Reliability Engineering
              * Google Cloud Platform (GCP)
              * Change Data Capture (CDC)
              * Bases de données
              * Performance
              * Benchmark
              * Production
              * Datadog
              * Modélisation

        - title: Contributeur Opensource
          company: Opensource
          company_url: ''
          company_logo:
          location: Valence (Remote)
          date_start: '2015-01-01'
          date_end: ''
          description: |2-
              Contributions à des projets opensource, notamment :

               * PostgreSQL : patch, relecture, remontée de bug
               * Powa : tests, hébergement et maintient de la plateforme de démo et de dev : https://demo-powa.anayrat.info & https://dev-powa.anayrat.info
               * check_pgactivity : tests, patchs
               * netdata : tests, patchs
               * pg_sampletolog : auteur
               * pgcheetah : auteur

              Compétences:

              * PostgreSQL
              * SQL
              * Optimisation des requêtes
              * Linux
              * Bases de données
              * Performance
              * Benchmark
              * Ansible
              * Modélisation
              * Opensource
              * Administration système Linux
              * Go
              * C

        - title: Expert PostgreSQL
          company: PeopleDoc
          company_url: ''
          company_logo:
          location: Valence (Remote)
          date_start: '2019-09-01'
          date_end: '2021-06-01'
          description: |2-
              DBA de production qui comprend administration des bases :
              * full text search
              * écriture de fonctions en pl/pgsql
              * partitionnement
              * écriture de migrations blue-green
              * optimisation/tuning
              * automatisation avec ansible
              * astreintes
              * veille technique.

              Compétences:

              * PostgreSQL
              * SQL
              * Optimisation des requêtes
              * Database Reliability Engineering
              * Linux
              * Change Data Capture (CDC)
              * pgBouncer
              * Bases de données
              * Performance
              * Benchmark
              * Production
              * Datadog
              * Modélisation
              * Infrastructure
              * Administration système Linux
              * Gestion d’incidents majeurs

        - title: Expert PostgreSQL
          company: Doctolib
          company_url: ''
          company_logo:
          location: Valence (Remote)
          date_start: '2018-07-01'
          date_end: '2019-05-01'
          description: |2-
              Maintient en condition opérationnelle de la base de donnée sous PostgreSQL à fort trafic (3.5To - 40K-50K TPS) :
              * Tuning système d'exploitation et PostgreSQL
              * Tests de charge
              * Supervision et métrologie
              * Optimisation de requêtes
              * Partage de connaissances auprès des développeurs et équipe DEVOPS afin d'exploiter au mieux le moteur
              * Veille technique sur PostgreSQL, Linux et hardware (CPU, stockage, mémoire)

              Compétences:

              * PostgreSQL
              * SQL
              * Optimisation des requêtes
              * Database Reliability Engineering
              * Linux
              * Bases de données
              * Performance
              * Benchmark
              * Production
              * Datadog
              * Modélisation
              * Infrastructure
              * Administration système Linux
              * Gestion d’incidents majeurs
              * Go
              * C


        - title: Consultant Expert PostgreSQL
          company: Dalibo
          company_url: ''
          company_logo:
          location: Valence (Remote)
          date_start: '2015-07-01'
          date_end: '2018-05-01'
          description: |2-
              Expert PostgreSQL:
              *  DBA Production
              *  Support
              *  Audit : Configuration, réplication, supervision
              *  Haute disponibilité
              *  Optimisation / tuning : SQL, PostgreSQL, Linux, stockage
              *  Contributions dans PostgreSQL : remontée de bugs, relecture de patch
              *  Formations
              *  Conférences : Orateur lors des pgday France
              *  Veille technique : PostgreSQL, Linux, virtualisation, stockage SAN

              Compétences:

              * PostgreSQL
              * SQL
              * Optimisation des requêtes
              * Linux
              * Bases de données
              * Production
              * pgBackRest
              * Modélisation
              * Infrastructure
              * Administration système Linux
              * Virtualisation
              * C

        - title: Ingénieur réseau et systèmes
          company: Axess Groupe
          company_url: ''
          company_logo:
          location: Valence
          date_start: '2013-07-01'
          date_end: '2015-06-01'
          description: |2-
              Maintient opérationnel et ingénierie de la plateforme "la-vie-scolaire.fr";. Solution d’Établissement Numérique de Travail pour Collèges et Lycées.
              * DBA Postgresql (optimisation, réplication)
              * Virtualisation vSphère 5.1/5.5
              * Tomcat 7
              * Métrologie (Collectd, ELK, Grafana)
              * Hébergement dédié/mutualisé : sites web, serveurs clients
              * FAI ADSL/SDSL/Fibre

              Compétences:

              * PostgreSQL
              * SQL
              * Linux
              * Bases de données
              * Production
              * Infrastructure
              * Administration système Linux
              * Virtualisation
              * IPv6
              * VMware ESX
              * Ansible

        - title: Gestion de projet - Orange
          company: NeoSoft
          company_url: ''
          company_logo:
          location: Rennes
          date_start: '2012-12-01'
          date_end: '2013-07-01'
          description: |2-
              Extension du réseau RAEI (réseau backbone offrant des services aux entreprises en France)
              * Pilotage du déploiement des routeurs (Juniper)
              * Pilotage de la mise en place des liens de transmissons

              Compétences:

              * Gestion de projet
              * Réseau

        - title: Ingénieur réseau
          company: NeoSoft
          company_url: ''
          company_logo:
          location: Rennes
          date_start: '2012-06-01'
          date_end: '2012-12-01'
          description: |2-
              Mission au Centre National de Traitement des radars automatiques
              * Administration switch, concentrateur VPN et routeurs (cisco, H3C)
              * Administration firewall (cisco ASA, BSD)
              * Failover, VRRP, CARP
              * Scripting bash

              En agence :
              * Présentation technique sur le protocole IPv6

              Compétences:

              * Linux
              * Production
              * Infrastructure
              * Réseau
              * IPv6
              * Virtualisation
        - title: Ingénieur réseaux (orange)
          company: SII
          company_url: ''
          company_logo:
          location: Rennes
          date_start: '2012-01-01'
          date_end: '2012-05-01'
          description: |2-
              Ingénieur Réseau chez Orange Buisness Services Technical stream leader, participe aux projets buisness VPN international, ingénierie et étude d'architecture réseau
              * Dimensionnement d'offres (VPN et VoIP)
              * Coordination avec les équipes d'ingénierie Voix et réseaux
              * Orientation sur les choix techniques (Réseaux, Voix)

              Compétences:

              * Linux
              * Production
              * Infrastructure
              * Administration système Linux
              * Virtualisation
        - title: Ingénieur sécurité (Orange Business Services)
          company: SII
          company_url: ''
          company_logo:
          location: Rennes
          date_start: '2011-03-01'
          date_end: '2011-12-01'
          description: |2-
              Chez Orange Buisness Services
              * Authentification forte avec RSA Securid et Active Identity
              * Virtualisation d'une solution de sécurité existante sous Vmware 4.1
              * Support N3
              * Radius : Juniper Steel Belted Radius
              * Rédaction de spec, documents d'exploitation (anglais)
              Compétences:

              * Linux
              * Production
              * Infrastructure
              * Administration système Linux
              * Virtualisation
              * VMware
        - title: Ingénieur système (orange)
          company: SII
          company_url: ''
          company_logo:
          location: Rennes
          date_start: '2009-10-01'
          date_end: '2011-03-01'
          description: |2-
              Ingénieur Système chez Orange (DEPFS). Exploitation des serveurs pour la VoIP (SIP).
              * Systèmes d'exploitation : Linux/solaris
              * Analyse trafic réseau : Wireshark
              * Protocole SIP
              * Plateforme de production Ericsson SIP permettant de prendre en charge plusieurs millions de clients
              * Installation de serveurs
              * Résolutions d'incident (support niveau 2) + astreintes
              * Expertise DNS/DHCP

              Compétences:

              * Linux
              * Production
              * Infrastructure
              * Administration système Linux
              * Virtualisation
        - title: Ingénieur apprenti
          company: Brest Métropole Océane
          company_url: ''
          company_logo:
          location: Brest
          date_start: '2006-09-01'
          date_end: '2009-09-01'
          description: |2-
              * Projet de fin d'étude sur le déploiement d'IPv6
              * Configuration de SAN (Création de LUN, Zoning sur switch Brocade...)
              * Installation Plateforme de supervision NAGIOS/Centreon
              * Installation et configuration d'un firewall de messagerie en Cluster (Borderware Security Platform 8.1)

              Compétences:

              * Linux
              * Virtualisation
              * VMware


    design:
      columns: '2'
  #- block: accomplishments
  #  content:
  #    # Note: `&shy;` is used to add a 'soft' hyphen in a long heading.
  #    title: 'Accomplish&shy;ments'
  #    subtitle:
  #    # Date format: https://wowchemy.com/docs/customization/#date-format
  #    date_format: Jan 2006
  #    # Accomplishments.
  #    #   Add/remove as many `item` blocks below as you like.
  #    #   `title`, `organization`, and `date_start` are the required parameters.
  #    #   Leave other parameters empty if not required.
  #    #   Begin multi-line descriptions with YAML's `|2-` multi-line prefix.
  #    items:
  #      - certificate_url: https://www.coursera.org
  #        date_end: ''
  #        date_start: '2021-01-25'
  #        description: ''
  #        organization: Coursera
  #        organization_url: https://www.coursera.org
  #        title: Neural Networks and Deep Learning
  #        url: ''
  #      - certificate_url: https://www.edx.org
  #        date_end: ''
  #        date_start: '2021-01-01'
  #        description: Formulated informed blockchain models, hypotheses, and use cases.
  #        organization: edX
  #        organization_url: https://www.edx.org
  #        title: Blockchain Fundamentals
  #        url: https://www.edx.org/professional-certificate/uc-berkeleyx-blockchain-fundamentals
  #      - certificate_url: https://www.datacamp.com
  #        date_end: '2020-12-21'
  #        date_start: '2020-07-01'
  #        description: ''
  #        organization: DataCamp
  #        organization_url: https://www.datacamp.com
  #        title: 'Object-Oriented Programming in R'
  #        url: ''
  #  design:
  #    columns: '2'
  # - block: collection
  #   id: posts
  #   content:
  #     title: Articles récents
  #     subtitle: ''
  #     text: ''
  #     # Choose how many pages you would like to display (0 = all pages)
  #     count: 5
  #     # Filter on criteria
  #     filters:
  #       folders:
  #         - post
  #       author: ""
  #       category: ""
  #       tag: ""
  #       exclude_featured: false
  #       exclude_future: false
  #       exclude_past: false
  #       publication_type: ""
  #     # Choose how many pages you would like to offset by
  #     offset: 0
  #     # Page order: descending (desc) or ascending (asc) date.
  #     order: desc
  #     archive:
  #       enable: true
  #       text: Voir tous les articles
  #   design:
  #     # Choose a layout view
  #     view: compact
  #     columns: '2'
  # - block: portfolio
  #   id: projects
  #   content:
  #     title: Evènements
  #     filters:
  #       folders:
  #         - project
  #     # Default filter index (e.g. 0 corresponds to the first `filter_button` instance below).
  #     default_button_index: 0
  #     # Filter toolbar (optional).
  #     # Add or remove as many filters (`filter_button` instances) as you like.
  #     # To show all items, set `tag` to "*".
  #     # To filter by a specific tag, set `tag` to an existing tag name.
  #     # To remove the toolbar, delete the entire `filter_button` block.
  #     buttons:
  #       - name: All
  #         tag: '*'
  #       - name: Deep Learning
  #         tag: Deep Learning
  #       - name: Other
  #         tag: Demo
  #   design:
  #     # Choose how many columns the section has. Valid values: '1' or '2'.
  #     columns: '1'
  #     view: showcase
  #     # For Showcase view, flip alternate rows?
  #     flip_alt_rows: false
  # - block: markdown
  #   content:
  #     title: Gallery
  #     subtitle: ''
  #     text: |-
  #       {{< gallery album="demo" >}}
  #   design:
  #     columns: '1'
  # - block: collection
  #   id: featured
  #   content:
  #     title: Featured Publications
  #     filters:
  #       folders:
  #         - publication
  #       featured_only: true
  #   design:
  #     columns: '2'
  #     view: card
  # - block: collection
  #   content:
  #     title: Recent Publications
  #     text: |-
  #       {{% callout note %}}
  #       Quickly discover relevant content by [filtering publications](./publication/).
  #       {{% /callout %}}
  #     filters:
  #       folders:
  #         - publication
  #       exclude_featured: true
  #   design:
  #     columns: '2'
  #     view: citation
  # - block: collection
  #   id: talks
  #   content:
  #     title: Conférences
  #     count: 4
  #     filters:
  #       folders:
  #         - event
  #   design:
  #     columns: '2'
  #     view: compact
  # - block: tag_cloud
  #   content:
  #     title: Sujets populaires
  #   design:
  #     columns: '2'
  - block: contact
    id: contact
    content:
      title: Contact
      subtitle:
      # text: |-
      #   Lorem ipsum dolor sit amet, consectetur adipiscing elit. Nam mi diam, venenatis ut magna et, vehicula efficitur enim.
      # Contact (add or remove contact options as necessary)
      email: adrien.nayrat@gmail.com
      # phone: 888 888 88 88
      # appointment_url: 'https://calendly.com'
      address:
        # street: 450 Serra Mall
        city: Valence
        region: FR
        postcode: '26000'
        country: France
        country_code: FR
      #directions: Enter Building 1 and take the stairs to Office 200 on Floor 2
      #office_hours:
      #  - 'Monday 10:00 to 13:00'
      #  - 'Wednesday 09:00 to 10:00'
      contact_links:
        # - icon: twitter
        #   icon_pack: fab
        #   name: Retrouvez-moi sur Twitter
        #   link: 'https://twitter.com/Adrien_nayrat'
        #- icon: skype
        #  icon_pack: fab
        #  name: Skype Me
        #  link: 'skype:echo123?call'
        #- icon: video
        #  icon_pack: fas
        #  name: Zoom Me
        #  link: 'https://zoom.com'
      # Automatically link email and phone or display as text?
      autolink: true
      # Email form provider
      # form:
      #   provider: netlify
      #   formspree:
      #     id:
      #   netlify:
      #     # Enable CAPTCHA challenge to reduce spam?
      #     captcha: false
      coordinates:
        latitude: '44.9347'
        longitude: '4.8917'
    design:
      columns: '2'
---
