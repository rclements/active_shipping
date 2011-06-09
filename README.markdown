# Active Shipping

This library is meant to interface with the web services of various shipping carriers. The goal is to abstract the features that are most frequently used into a pleasant and consistent Ruby API. Active Shipping is an extension of [Active Merchant][], and as such, it borrows heavily from conventions used in the latter.

We are starting out by only implementing the ability to list available shipping rates for a particular origin, destination, and set of packages. Further development could take advantage of other common features of carriers' web services such as tracking orders and printing labels.

Active Shipping is currently being used and improved in a production environment for the e-commerce application [Shopify][]. Development is being done by [James MacAulay][] (<james@jadedpixel.com>). Discussion is welcome in the [Active Merchant Google Group][discuss].

[Active Merchant]:http://www.activemerchant.org
[Shopify]:http://www.shopify.com
[James MacAulay]:http://jmacaulay.net
[discuss]:http://groups.google.com/group/activemerchant

## Supported Shipping Carriers

* [UPS](http://www.ups.com)
* [USPS](http://www.usps.com)
* [FedEx](http://www.fedex.com)
* [Canada Post](http://www.canadapost.ca)
* [New Zealand Post](http://www.nzpost.co.nz)
* more soon!

## Prerequisites

* [active_support](http://github.com/rails/rails/tree/master/activesupport)
* [xml_node](http://github.com/tobi/xml_node/) (right now a version of it is actually included in this library, so you don't need to worry about it yet)
* [mocha](http://mocha.rubyforge.org/) for the tests

## Download & Installation

Currently this library is available on GitHub:

  <http://github.com/Shopify/active_shipping>

You will need to get [Git][] if you don't have it. Then:

  > git clone git://github.com/Shopify/active_shipping.git

(That URL is case-sensitive, so watch out.)
  
Active Shipping includes an init.rb file. This means that Rails will automatically load it on startup. Check out [git-archive][] for exporting the file tree from your repository to your vendor directory.

[Git]:http://git.or.cz/
[git-archive]:http://www.kernel.org/pub/software/scm/git/docs/git-archive.html

## Sample Usage

### Compare rates from different carriers

    require 'active_shipping'
    include ActiveMerchant::Shipping
  
    # Package up a poster and a Wii for your nephew.
    packages = [
      Package.new(  100,                        # 100 grams
                    [93,10],                    # 93 cm long, 10 cm diameter
                    :cylinder => true),         # cylinders have different volume calculations
    
      Package.new(  (7.5 * 16),                 # 7.5 lbs, times 16 oz/lb.
                    [15, 10, 4.5],              # 15x10x4.5 inches
                    :units => :imperial)        # not grams, not centimetres
    ]
  
    # You live in Beverly Hills, he lives in Ottawa
    origin = Location.new(      :country => 'US',
                                :state => 'CA',
                                :city => 'Beverly Hills',
                                :zip => '90210')
  
    destination = Location.new( :country => 'CA',
                                :province => 'ON',
                                :city => 'Ottawa',
                                :postal_code => 'K1P 1J1')
                              
    # Find out how much it'll be.
    ups = UPS.new(:login => 'auntjudy', :password => 'secret', :key => 'xml-access-key')
    response = ups.find_rates(origin, destination, packages)
  
    ups_rates = response.rates.sort_by(&:price).collect {|rate| [rate.service_name, rate.price]}
    # => [["UPS Standard", 3936],
    #     ["UPS Worldwide Expedited", 8682],
    #     ["UPS Saver", 9348],
    #     ["UPS Express", 9702],
    #     ["UPS Worldwide Express Plus", 14502]]
  
    # Check out USPS for comparison...
    usps = USPS.new(:login => 'developer-key')
    response = usps.find_rates(origin, destination, packages)
  
    usps_rates = response.rates.sort_by(&:price).collect {|rate| [rate.service_name, rate.price]}
    # => [["USPS Priority Mail International", 4110],
    #     ["USPS Express Mail International (EMS)", 5750],
    #     ["USPS Global Express Guaranteed Non-Document Non-Rectangular", 9400],
    #     ["USPS GXG Envelopes", 9400],
    #     ["USPS Global Express Guaranteed Non-Document Rectangular", 9400],
    #     ["USPS Global Express Guaranteed", 9400]]
    
### Track a FedEx package

    fedex = FedEx.new(:login => '999999999', :password => '7777777')
    tracking_info = fedex.find_tracking_info('tracking-number', :carrier_code => 'fedex_ground') # Ground package
    
    tracking_info.shipment_events.each do |event|
      puts "#{event.name} at #{event.location.city}, #{event.location.state} on #{event.time}. #{event.message}"
    end
    # => Package information transmitted to FedEx at NASHVILLE LOCAL, TN on Thu Oct 23 00:00:00 UTC 2008. 
    # Picked up by FedEx at NASHVILLE LOCAL, TN on Thu Oct 23 17:30:00 UTC 2008. 
    # Scanned at FedEx sort facility at NASHVILLE, TN on Thu Oct 23 18:50:00 UTC 2008. 
    # Departed FedEx sort facility at NASHVILLE, TN on Thu Oct 23 22:33:00 UTC 2008. 
    # Arrived at FedEx sort facility at KNOXVILLE, TN on Fri Oct 24 02:45:00 UTC 2008. 
    # Scanned at FedEx sort facility at KNOXVILLE, TN on Fri Oct 24 05:56:00 UTC 2008. 
    # Delivered at Knoxville, TN on Fri Oct 24 16:45:00 UTC 2008. Signed for by: T.BAKER

### Generating a UPS Label

	require 'base64'
	require 'fileutils'
	require 'tempfile'
	require 'active_shipping'

	class UpsLabelTest
  
	  include ActiveMerchant::Shipping
  
	  UPS_ORIGIN_NUMBER = "111111"
	  UPS_LOGIN = 'xxxxxxx'
	  UPS_PASSWORD = 'zzzzzzz'
	  UPS_KEY = '0000000000000'
  
	  TESTING = true
	  SAVE_LABEL_LOCATION = "/path/to/save/test/labels/here"
  
	    # services located in UPS::DEFAULT_SERVICES
	    #
	    # "01" => "UPS Next Day Air",
	    # "02" => "UPS Second Day Air",
	    # "03" => "UPS Ground",
	    # "07" => "UPS Worldwide Express",
	    # "08" => "UPS Worldwide Expedited",
	    # "11" => "UPS Standard",
	    # "12" => "UPS Three-Day Select",
	    # "13" => "UPS Next Day Air Saver",
	    # "14" => "UPS Next Day Air Early A.M.",
	    # "54" => "UPS Worldwide Express Plus",
	    # "59" => "UPS Second Day Air A.M.",
	    # "65" => "UPS Saver",
	    # "82" => "UPS Today Standard",
	    # "83" => "UPS Today Dedicated Courier",
	    # "84" => "UPS Today Intercity",
	    # "85" => "UPS Today Express",
	    # "86" => "UPS Today Express Saver"
  
  
	  def initialize
	    @ups = UPS.new(:login => UPS_LOGIN, :password => UPS_PASSWORD, :key => UPS_KEY)
	  end #end method initialize
  
  
	  def run_tests
	    #UPS::DEFAULT_SERVICES.keys.sort.each do |code|
	    ["03"].each do |code|
	      begin
	        trk_num =  run_test_for_service(code)
	        puts "Generated label for: #{UPS::DEFAULT_SERVICES[code]} => #{trk_num}"
	      rescue => e
	        puts "ERROR GENERATING: #{UPS::DEFAULT_SERVICES[code]} => #{e.message}"
	      end
	    end
	  end #end method run_tests
  
  
	  def get_packages
	    #please refer Package class (lib/shipping/package.rb) for more info
	    [
	      Package.new((3 * 16),[12, 12, 12], :units => :imperial)
	    ]
	  end #end method get_packages
  
  
	  def get_label_specification
	    # Label print method code that the labels are to be generated for EPL2 formatted 
	    # labels use EPL, for SPL formatted labels use SPL, for ZPL formatted labels use ZPL, 
	    # for STAR printer formatted labels use STARPL and for image formats use GIF.
    
	    {:print_code => "GIF", :format_code => "GIF", :user_agent => "Mozilla/4.5"}
	  end #end method label_specification
  
  
	  def get_options
	    #create a options hash containing origin, destination. For test environment pass :test => true
	    options = {
	      :origin => {
	        :address_line1 => "1 Arena Plaza", 
	        :country => 'US', 
	        :state => 'KY',
	        :city => 'Louisville',
	        :zip => '40202', 
	        :phone => "(502) 555-1212", 
	        :name => "My Destination Name", 
	        :attention_name => "Receiving Department", 
	        :origin_number => UPS_ORIGIN_NUMBER
	      }, 
	      :destination => {
	        :company_name => "Acme Co. Ltd.",
	        :attention_name => "John Smith",
	        :phone => "(555) 555-5555", 
	        :address_line1 => "1234 No Street", 
	        :country => 'US', 
	        :state => 'KY', 
	        :city => 'Louisville', 
	        :zip => '40202'
	        }, 
	      :test => TESTING
	    }

	  end #end method get_options
  
  
	  def create_confirm_response(carrier_service, packages, label_specification, options)
	    #send the Shipment Confirm Request and catch the response. if successful then it will return an identification number, shipment charges and a shipment digest.
	    @confirm_response = @ups.shipment_confirmation_request(carrier_service, packages, label_specification, options)
	  end #end method create_confirm_response
  
  
	  def create_shipment_request(confirm_response)
	    #send the Shipment Accept Request and catch the response. if successful then it will return tracking number, base64 html label, base64 graphic label for each package and identification number for the shipment.
	    @accept_response = @ups.shipment_accept_request(confirm_response.digest, {:test => TESTING})
	  end #end method create_shipment_request(confirm_response)
  
  
  
  
	  def run_test_for_service(carrier_service)
    
	    packages = get_packages
    
	    label_specification = get_label_specification
  
	    options = get_options

	    confirm_response = create_confirm_response(carrier_service, packages, label_specification, options)

	    accept_response = create_shipment_request(confirm_response)

	    out = []

	    #To get label and other info of each package of the above shipment
	    accept_response.shipment_packages.each do |package|
      
	      #gives you the base64 code for html label
	    	html_image = package.html_image 
    	
	    	#gives you the base64 code for graphic label
	    	graphic_image = package.graphic_image 
    	
	    	#gives you the images format(gif/png)
	    	label_image_format = package.label_image_format 
    	
	    	#gives you the tracking number of package
	    	tracking_number = package.tracking_number 
    	
    	
	    	#write out the GRAPHIC file
	    	label_tmp_file = Tempfile.new("shipping_label")
	      label_tmp_file.write Base64.decode64(graphic_image)
	      label_tmp_file.rewind
      
	      #write out the HTML file
	      html_tmp_file = Tempfile.new("shipping_label_html")
	      html_tmp_file.write Base64.decode64(html_image)
	      html_tmp_file.rewind
      
	      #save the GRAPHIC file
	      graphic_filename = "#{SAVE_LABEL_LOCATION}/label#{tracking_number}.#{label_image_format.downcase}"
	      gf = File.new(graphic_filename, "wb")
	      gf.write File.new(label_tmp_file.path).read
	      gf.close
      
	      #save the HTML file
	      html_filename = "#{SAVE_LABEL_LOCATION}/#{tracking_number}.html"
	      hf = File.new(html_filename, "wb")
	      hf.write File.new(html_tmp_file.path).read
	      hf.close
 
	    	out << tracking_number
    	
	    end #end shipment_packages.each
    
	    return out
	  end #end run_test_for_service
  
	end #end class
	
	
	test = UpsLabelTest.new
	test.run_tests # <== Now go check your folder

## TODO

* proper documentation
* carrier code template generator
* more carriers
* support more features for existing carriers
* bin-packing algorithm (preferably implemented in ruby)
* order tracking
* label printing

## Contributing

Yes, please! Take a look at the tests and the implementation of the Carrier class to see how the basics work. At some point soon there will be a carrier template generator along the lines of the gateway generator included in Active Merchant, but carrier.rb outlines most of what's necessary. The other main classes that would be good to familiarize yourself with are Location, Package, and Response.

The nicest way to submit changes would be to set up a GitHub account and fork this project, then initiate a pull request when you want your changes looked at. You can also make a patch (preferably with [git-diff][]) and email to james@jadedpixel.com.

[git-diff]:http://www.kernel.org/pub/software/scm/git/docs/git-diff.html

## Contributors

* James MacAulay (<http://jmacaulay.net>)
* Tobias Luetke (<http://blog.leetsoft.com>)
* Cody Fauser (<http://codyfauser.com>)
* Jimmy Baker (<http://jimmyville.com/>)
* William Lang (<http://williamlang.net/>)
* Brian Webb (<http://parkersmithsoftware.com>)

## Legal Mumbo Jumbo

Unless otherwise noted in specific files, all code in the Active Shipping project is under the copyright and license described in the included MIT-LICENSE file.