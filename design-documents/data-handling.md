## Data handling


### Terms

Term|Meaning
----|-------
Data|Any form of input or information from anywhere. Includes but not limited to: browser form submissions, REST request, CSV import, internal information movement such as conversion/transformation/adaptation between different types. 
Validation|The process of asserting that data is valid within a specific context for a specific purpose.
Transformation|The process of filtering, sanitizing, normalizing, or otherwise modifying data in preparation to be used in a specific context for a specific purpose. Commonly used to prepare data for validation but could be used for other mutations as well.
Input Context|The context in which data is obtained. Includes but not limited to: browser form submission, CSV import, graphql mutation, REST request, entity data hydration or deserialization
Desired Context|The context for which the data is intended to be used in. Includes but not limited to: database row, serialized string,
Input Filter|A mechanism that can Transform and Validate data for its Desired Context using configuration specific to the Input Context

### Overview

**TL;DR**: Magento needs to have defined behavior for data handling (at rest, in transit, and in processing) and add native support for it as a necessary step to prevent product rot especially as security becomes increasingly more important.

**High level problem**: Magento currently has essentially no useful guidelines or framework-level support for data handling. 
Currently data MAY be handled on a case-by-case basis but there isn't a process that MUST be followed or even a defined workflow that MAY be followed. In addition, as described in the wiki page referenced below, the technical vision isn't clear in this matter and the code throughout the codebase old and new also varies in what practices are adopted. 
 
**An example**: To illustrate a concrete example of why this is such a problem let's look at CMS pages. 
CMS pages may be created via rest/soap, graphql (eventually), CSV import, and of course web admin. 
Each of these mechanisms are completely separate from each other and each one has a separate method of handling input sanitization, validation, transformation, and persistence. 
As a result, each of these aspects are handled totally different with each Input Context despite the data (at the core of the model) being the same.

**Further damage**: To make this even worse we have the same problem with the persistence layer. 
Data is processed differently depending on whether the ResourceModel or Repository (or both in some cases) is being used. 
To make this even worse we also tend to violate the interface contracts and depend on concrete implementation which leads to unclear dependency trees and even more unexpected handling of data. 

**Conclusion**: All of these things contribute to a very inconsistent, insecure, and undesirable experience for merchants, developers, product owners, and especially security experts across all of Magento and throughout the development community.

**Solution**: Implement an Input Filter in Framework that is natively supported by core resource models to, at very least, provide a level of base consistency of validation.

**References**
* [This wiki page](https://wiki.corp.magento.com/pages/viewpage.action?spaceKey=MAGE2&title=Proposal%3A+Data+Validation+Unification) explains that even our current technical vision is flawed and conflicts with large amounts of existing code in this area. It also reviews 6 of the top validation libraries and their compatibility with our Technical guidelines as well as the defined requirements (on the wiki page) for the chosen validation library. Selecting a good library would assist in the fulfillment of this proposal but it only covers a small portion of what this proposal hopes to achieve.  

### Design

Aside from the primary purpose established in the rest of this document, the goal for the initial design and base implementation is to create a mechanism within the Model system that can serve the Desired Context for data at rest per model.
  
The base implementation SHOULD allow for the Input Context and Desired Context to be configured per invocation or use. The integration of this framework in Model as described above could look at which `\Magento\Framework\App\Area::AREA_*` is currently set with `\Magento\Framework\App\State::getAreaCode` to select a default Input Context but MUST be able to be overridden per use as the assumption will not always be correct.

Due to the Technical Vision violations described in the wiki page mentioned, it does not seem appropriate to incorporate `Magento\Framework\Validator\*` validators as-is. The biggest conflict is that the primary interface `Magento\Framework\Validator\ValidatorInterface` extends `Zend_Validate_Interface`. Regardless of which direction below is chosen, a new interface that mirrors `Zend_Validate_Interface` within `Magento\Validator` should be created.

I see two ways to support the existing validators with further coupling incorrect interfaces while also supporting backwards compatibility. 
* Option1: Add the new interface (`implements NewValidatorInterace`) to all of the current classes that implement the incorrect `Zend_Validate_Interface`. 
* Option2: Create a generic adapter that implements only the new interface but accepts and calls the methods of a `Zend_Validate_Interface`. 


TODO: Super awesome design and charts and stuff TBD

#### Acceptance Criteria Fulfillment

1. MUST be compatible with MINOR release
1. Components within the InputFilter tree MAY contain state but MUST be idempotent and therefore reset the state for each usage. For example, a validator MUST be able to be called more than once using different data sets without any issues. 
1. Exceptions MUST be handled according the Technical Vision and only used for unexpected behavior. Given that the domain of these components are for validating, it is not unexpected if the data is invalid. Error messages should be used instead. 
1. MUST consider how `Magento\Framework\Validator\*` can be incorporated. 
1. Input Filter MUST vary Validation and Transformation for Desired Context based on Input Context
1. Abstraction API layer MUST be implemented in `Magento\Framework` to allow for maximum compatibility
1. Implementations MUST be able to rely on nothing more than the API abstraction layer
1. Concrete implementations of the abstraction layer SHOULD be irrelevant to most use cases
1. Initial design SHOULD be implemented in a way that allow Magento modules to use the abstraction layer in a backwards compatible way
1. Conditional configurations MUST be supported without hacky workarounds. e.g. validating a postal code is valid based on an associated country and state/province.   

#### Component Dependencies

There are no Magento dependencies required by the base implementation of this validation system as required by the Acceptance criteria. 

#### Extension Points and Scenarios

**Scenarios**:

1. **Basic validation**: Field A should be stripped of trailing spaces, not be empty and appear to be a valid URL.
1. **Contextual basic validation**: When coming from a web form field A should be converted to a constant if it meets some condition (e.g. equal to the string `true`) but processed as-is by the Input Filter if in any other context.
1. **Conditional validation**: Field A should only be validated if field B is not empty. Example: Dropdown of predefined options with the last option being "Other". When "Other" is chosen, an additional textbox should be processed.
1. **Dependent validation**: Field A should only be validated if field B and C meet certain conditions. Example: A dataset with Country, State, Postal Code, and Phone number. Postal code and Phone number both require a valid filtered value from their dependent fields (Country and State) in order to determine if the field is valid.
1. **Dependent conditional validation**: Fields C-Z should only be analyzed if field A and B meet certain conditions. Example: A checkbox that indicates a product is downloadable. Downloadable fields should now be analyzed. 

**Extension Points**

Yes, many. TBD.
 

### Prototype or Proof of Concept

Rough initial work-in-progress [in this repo](https://github.com/nathanjosiah/magento-data-framework/blob/master/Magento/Framework/InputFilter.php)

### Data size and Performance Requirements

No significant data size or performance implications are expected if implemented correctly aside from obvious processing time required that is inherent to added Validation and Transformation 

