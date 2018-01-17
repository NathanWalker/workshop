## Nx Challenge Part 2 - Refactor to share more

What we setup in Chapter 7 can be refactored to share more. Let's see if we can work together to reduce common code to our shared lib.

### Use "abstract" base classes for common component traits

The commonality between the web's `app.component.ts` (home view) and mobiles `items.component.ts` (home view) is almost identical so we can refactor those common traits to a base abstract class.

Our tasks for this chapter will be to work together to refactor as much as we can from what we have done to share more code. 

#### Bonus #1: use items list in the web as well

Mobile provides a list of futbol players. We can use that list on the web as well. How?

