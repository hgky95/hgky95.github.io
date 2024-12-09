---
title: 'Stateless Password Generator: Secure and Hassle-Free Password Management'
categories: [tools, security]
date: 2024-12-08 17:00:00
tags: [tools, todayilearned, TIL]
image: "/assets/img/stateless_pw_generator/password-manager.jpeg"
---

# Stateless Password Generator: Secure and Hassle-Free Password Management

Managing multiple passwords across various platforms can be daunting. The Stateless Password Generator simplifies this process using a secure, stateless Master Password algorithm. This tool eliminates the need to store passwords while ensuring robust security. It’s available for installation on the [Chrome Web Store](https://chromewebstore.google.com/detail/stateless-password-manage/cfmcfaddoadgpdnhbnofmbnhooigchlk), operating entirely offline for maximum privacy.

E.g: Generate passwords for Facebook

![ui](https://github.com/user-attachments/assets/3fee35ef-4058-4885-aed0-7ca888b9496e)

## Key Features

1. **Stateless Operation**: No data is stored, and passwords are generated dynamically using your master password.
2. **Customizable Preferences**: Adjust password settings, including length and character requirements (uppercase, lowercase, numbers, special characters).
3. **Offline Functionality**: No external connections are needed, enhancing security.
4. **Single Master Password**: Memorize one master password for all accounts, simplifying password management.

## How It Works

The Stateless Password Generator employs a cryptographic hash function to generate unique passwords for each website. The algorithm ensures the generated passwords adhere to the user-defined constraints, such as required character types and maximum length.

### Core Algorithm

Here’s a breakdown of the password generation process:

1. **User Input**:

   - Domain name
   - Username
   - Master password
   - Additional preferences (e.g., password length, required character types)

2. **Hashing**: The inputs are combined into a single string and hashed using the `SHA-256` algorithm. This ensures a unique and deterministic hash value for each set of inputs.

3. **Password Construction**:

   - Required character rules are extracted from the user’s preferences.
   - The hashed output is mapped to characters from defined sets (e.g., uppercase, lowercase, numbers, special characters).
   - The resulting password satisfies all constraints and is truncated to the specified length.

### Code Highlights

Below are key functions that power the Stateless Password Generator:

#### Define Character Sets

```javascript
const upperChars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
const lowerChars = 'abcdefghijklmnopqrstuvwxyz';
const numberChars = '0123456789';
const specialChars = '!@#$%^&*()';
const CHARACTER_SETS = {
    uppercase: upperChars,
    lowercase: lowerChars,
    number: numberChars,
    special: specialChars,
};
```

#### Hashing Logic

```javascript
async function hashPassword(userData) {
    const combinedString = userData.domain + userData.username + userData.masterPassword + userData.pwVersion;
    const encoder = new TextEncoder();
    const passwordHash = await crypto.subtle.digest('SHA-256', encoder.encode(combinedString));
    const passwordHashArray = Array.from(new Uint8Array(passwordHash));

    const allRequiredChars = getRequireChars(getRequireRules(
        userData.isRequiredUpperCase,
        userData.isRequiredLowerCase,
        userData.isRequiredNumber,
        userData.isRequiredSpecial
    ));

    let password = "";
    for (let i = 0; i < userData.maxLength; i++) {
        let byte = passwordHashArray[i % passwordHashArray.length];
        password += allRequiredChars[byte % allRequiredChars.length];
    }
    return password;
}
```

#### Character Rule Mapping

```javascript
function getRequireRules(isRequiredUpperCase, isRequiredLowerCase, isRequiredNumber, isRequiredSpecial) {
    let rules = [];
    if (isRequiredUpperCase) rules.push('uppercase');
    if (isRequiredLowerCase) rules.push('lowercase');
    if (isRequiredNumber) rules.push('number');
    if (isRequiredSpecial) rules.push('special');
    return rules;
}
```

## Benefits

- **Enhanced Security**: Passwords are generated locally and never transmitted or stored.
- **Convenience**: Only remember your master password to access all generated passwords.
- **Customizable**: Tailor password generation to meet stringent security requirements.

## Conclusion

The Stateless Password Generator is a powerful tool for managing passwords securely and efficiently. By leveraging cryptographic hashing and stateless algorithms, it offers robust protection without compromising usability. Install it from the [Chrome Web Store](https://chromewebstore.google.com/detail/stateless-password-manage/cfmcfaddoadgpdnhbnofmbnhooigchlk)!

Please checkout the [GitHub](https://github.com/hgky95/password-manager) for more details.

Enjoying the project? Don’t forget to star it <a href="https://github.com/hgky95/password-manager">Star me here ⭐ !</a>

