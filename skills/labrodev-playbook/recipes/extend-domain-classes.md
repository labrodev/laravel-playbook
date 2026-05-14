# Extend domain classes

According to playbook documents and stubs, and Laravel conventions, fill up all other domain classes (minimal which need to provide create,update,remove and read objects of models).

And for Data classes validations use from schema data types and requirements. Use enums if you find it's good (but ask me), use casts.

Add it for all domains for all models (with all necessary fields in Data classes and Create/Update fields).

Keep in mind that you need to provide Enums for statuses and other corresponding fields (according to docs in playbook).

Also keep in mind that in actions you need to set up values for model attributes from data classes without checking if they are null or not. in cases if you need to assign id from casted object you may use ternary operator (example: $objectData->anotherObject->id ?? null).

Read schema sql database and use all fields in the corresponding data types and nullable/required flags there. Apply validations.

Wire each model’s observer with `#[ObservedBy]` on the model class (see `stubs/core/domain/models/model.stub`).
Add Rules classes.
Add Policies.
Add Factories.

Do all sort of necessary things according to structure and philosophy and stubs which are all described for you in playbook.
