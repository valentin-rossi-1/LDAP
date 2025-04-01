
![enter image description here](https://www.okta.com/sites/default/files/styles/1640w_scaled/public/media/image/2021-03/how-ldap-works.png?itok=KPHB6M5H)

# **Job 1: Understanding and Preparing LDAP**

### **1. Introduction to LDAP**

-   **What is LDAP?**  
    LDAP (Lightweight Directory Access Protocol) is an open-source protocol used to access and manage directory services. It allows storing, organizing, and retrieving information about users, groups, and resources within a network.
    
-   **Usage in companies**:
    
    -   Centralized user and permission management.
    -   Simplified authentication (SSO – Single Sign-On).
    -   Access management for shared resources (files, applications, etc.).
-   **Advantages of centralization**:
    
    -   Reduces errors and duplicates.
    -   Improves security through centralized password and policy management.
    -   Eases maintenance and information updates.

----------

### **2. Defining User Groups**

I categorize users into three groups based on their roles:

1.  **Basic Users**
    
    -   **Permissions**: Access to shared files.
    -   **Explanation**: These users can read and write in shared folders but do not have access to system settings or critical applications.
2.  **Developers**
    
    -   **Permissions**: Limited access to development applications (IDEs, code repositories, etc.).
    -   **Explanation**: They can modify and test code but cannot change security settings or system configurations.
3.  **Administrators**
    
    -   **Permissions**: Full rights, including modifying security settings.
    -   **Explanation**: These users have complete system access and can manage users, groups, and security policies.

----------

# **Job 2: Technical Implementation**

## **1. Installing LDAP**

I start by updating my system and installing OpenLDAP along with the necessary tools:

```bash
sudo apt update && sudo apt install slapd ldap-utils -y

```

During installation, I set a password for the LDAP administrator and make sure to save it securely.

If I need to reconfigure LDAP, I run the following command:

```bash
sudo dpkg-reconfigure slapd

```

Here are the choices I make:

-   **LDAP domain name**: `samuel.com` (which translates to `dc=samuel,dc=com`)
-   **Organization**: Samuel Inc
-   **Admin password**: I use the same one I set earlier
-   **MDB database**
-   **No** to deleting the existing database
-   **Yes** to moving old files

I then verify that the LDAP service is running correctly with:

```bash
sudo systemctl status slapd

```

----------

## **2. Creating Groups and Users**

### **2.1 Creating Groups**

I create a `groups.ldif` file with the necessary groups:

```ldif
dn: ou=groups,dc=samuel,dc=com
objectClass: organizationalUnit
ou: groups

dn: cn=user,ou=groups,dc=samuel,dc=com
objectClass: posixGroup
cn: Basique
gidNumber: 1001

dn: cn=developer,ou=groups,dc=samuel,dc=com
objectClass: posixGroup
cn: Developpeur
gidNumber: 1002

dn: cn=administrator,ou=groups,dc=samuel,dc=com
objectClass: posixGroup
cn: Administrateur
gidNumber: 1003

```

I add these groups to LDAP using the following command:

```bash
ldapadd -x -D "cn=admin,dc=samuel,dc=com" -W -f groups.ldif

```

----------

### **2.2 Creating Users**

I then create a `users.ldif` file to add my users to LDAP:

```ldif
dn: ou=users,dc=samuel,dc=com
objectClass: organizationalUnit
ou: users

dn: uid=alice,ou=users,dc=samuel,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: alice
sn: Alice
givenName: Alice
cn: Alice
displayName: Alice
uidNumber: 1001
gidNumber: 1001
homeDirectory: /home/alice
loginShell: /bin/bash
userPassword: {SSHA}hashedpassword

dn: uid=bob,ou=users,dc=samuel,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: bob
sn: Bob
givenName: Bob
cn: Bob
displayName: Bob
uidNumber: 1002
gidNumber: 1002
homeDirectory: /home/bob
loginShell: /bin/bash
userPassword: {SSHA}hashedpassword

dn: uid=claire,ou=users,dc=samuel,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: claire
sn: Claire
givenName: Claire
cn: Claire
displayName: Claire
uidNumber: 1003
gidNumber: 1003
homeDirectory: /home/claire
loginShell: /bin/bash
userPassword: {SSHA}hashedpassword

```

Before executing the command, I need to generate a hashed password with:

```bash
sudo slappasswd

```

I then replace `{SSHA}hashedpassword` in `users.ldif` with the generated value.

Finally, I add the users to LDAP:

```bash
ldapadd -x -D "cn=admin,dc=samuel,dc=com" -W -f users.ldif

```

----------

## **3. Access Verification**

To verify that everything is properly configured, I run an LDAP search:

```bash
ldapsearch -x -LLL -D "cn=admin,dc=samuel,dc=com" -W -b "dc=samuel,dc=com"

```

Then, I test user authentication. For example, for Alice:

```bash
ldapwhoami -x -D "uid=alice,ou=users,dc=samuel,dc=com" -W

```

If everything works correctly, the command should return:

```bash
dn:uid=alice,ou=users,dc=samuel,dc=com

```

----------

# **Job 3: Basic Security Options**

## **1. Password Management**

**Objective**: Configure LDAP to enforce a minimum password length of at least 8 characters.

**Steps:**

1.  **Install required modules**
    
    ```bash
    sudo apt install slapd slapd-contrib
    
    ```
    
2.  **Enable the `ppolicy` module**
    
    I create a file `ppolicy.ldif` and add the necessary configurations.
    
    Then, I load the module with:
    
    ```bash
    sudo ldapadd -Y EXTERNAL -H ldapi:/// -f ppolicy.ldif
    
    ```
    
3.  **Create a password policy**
    
    I create a `ppolicy.ldif` file and configure the password policy:
    ```bash
    dn: cn=default,ou=policies,dc=samuel,dc=com
    objectClass: person
    objectClass: top
    cn: default
    sn: default
    # Attribute that contains the password: userPassword
    pwdAttribute: userPassword
    # Minimum password length: 8 characters
    pwdMinLength: 8
    # Number of previous passwords stored to prevent reuse: 5
    pwdInHistory: 5
    pwdCheckQuality: 1
    # Maximum number of failed login attempts: 3
    pwdMaxFailure: 3
    # Locks the account if 3 consecutive login attempts fail
    pwdLockout: TRUE
    pwdLockoutDuration: 300
    pwdFailureCountInterval: 300
    pwdMustChange: TRUE
    # Allows the user to change their own password
    pwdAllowUserChange: TRUE
    pwdExpireWarning: 604800
    pwdGraceAuthNLimit: 5
    pwdSafeModify: FALSE
    ```
        
    I apply the policy with:
    
    ```bash
    sudo ldapadd -x -D "cn=admin,dc=samuel,dc=com" -W -f ppolicy.ldif
    
    ```
    
4.  **Verify configuration**
    
    ```bash
    ldapsearch -x -b "ou=policies,dc=samuel,dc=com" -D "cn=admin,dc=samuel,dc=com" -W
    
    ```
    

----------

## **2. Automatic Logout**

**Objective**: Implement automatic logout after a period of inactivity.

**Steps:**

1.  **Set inactivity timeout**
    
    I modify the password policy to add an inactivity timeout (`pwdIdleTimeout`).
    
    I create an `idle_timeout.ldif` file:
    
    ```ldif
    dn: cn=default,ou=policies,dc=samuel,dc=com
    changetype: modify
    add: pwdIdleTimeout
    pwdIdleTimeout: 600
    
    ```
    
    I apply the change with:
    
    ```bash
    sudo ldapmodify -x -D "cn=admin,dc=samuel,dc=com" -W -f idle_timeout.ldif
    
    ```
    
    Here, `600` means 600 seconds (10 minutes) of inactivity before logout.
    
2.  **Verify configuration**
    
    ```bash
    ldapsearch -x -b "ou=policies,dc=samuel,dc=com" -D "cn=admin,dc=samuel,dc=com" -W
    
    ```
