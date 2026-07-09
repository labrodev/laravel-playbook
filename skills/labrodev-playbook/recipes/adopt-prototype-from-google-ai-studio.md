# Adopt react prototype from google ai studio

## Task description

Your task: to implement react components and pages from prototype (which is simple google ai studio interface prototype) to an actual pages (with routes) and views and controllers in supplier application.

So in terms of relations between prototype pages and Laravel controllers (and routes) it should be:

1) BookingList, BookingDetails -> App\Dashboard\Booking\Controllers\BookingController
2) CustomerList, CustomerDetails -> App\Dasboard\Customer\Controllers\CustomerController
3) InterestList, InterestDetails -> App\Dashboard\Booking\Controllers\InterestController
4) CompanyInfo -> App\Dashboard\Supplier\Controllers\SupplierController
5) AreaList, AreaDetails -> App\Dashboard\Supplier\Controllers\AreaController
6) ScheduleSettings -> App\Dashboard\Configuration\Controllers\ConfigurationController
7) TimewaveSettings -> App\Dashboard\Supplier\Controllers\SupplierController
8) ServiceList, ServiceDetails -> App\Dashboard\Service\Controllers\ServiceController
9) DiscountList, DiscountDetails -> App\Dashboard\Supplier\Controllers\CampaignController
10) AggregatedData -> App\Dashboard\Integration\Controllers\CompanionController, App\Dashboard\Integration\Controllers\EmployeeController
11) Simulator -> App\Dashboard\Platform\Controllers\SimulatorController
12) Administration -> App\Dashboard\Platform\Controllers\AdministrationController
13) Profile -> App\Dashboard\Platform\Controllers\ProfileController
14) Dashboard homepage -> App\Dashboard\Platform\Controllers\HomeController
15) Health -> App\Dashboard\Platform\Controllers\HealthController

Should be used Inertia. Follow `docs/11-inertia-react.md` for all frontend structure, conventions, and boundaries.

## Prototype location

Prototypes are located in the '.. /prototype' folder. 

