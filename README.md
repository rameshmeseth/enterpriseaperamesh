Enterprisape Ramesh
====================


My first application
====================

  def initialize
    setup
  end

  protected

  def setup
    @browser = $active_browser
  end

  public

  # ------ Browser methods ------

  def navigate_to_site(site)
    $acp_site_url = site
    $active_browser.goto(site)
  end

  def focus_browser
    $active_browser.focus
  end

  def bring_browser_to_front
    $active_browser.window.use
  end

  def scroll_to_top
    $active_browser.scroll.to :top
    sleep 1
  end

  def scroll_to_center
    $active_browser.scroll.to :center
    sleep 1
  end

  def scroll_to_bottom
    $active_browser.scroll.to :bottom
    sleep 1
  end

  def scroll_into_view(tag, attribute_name, attribute_value)
    find_element(tag, attribute_name, attribute_value).wd.location_once_scrolled_into_view
  end

  def move_focus(x, y)
    $active_browser.driver.executeScript(0, 400)
  end

  def sleep_by_browser_type(sleep_amount)
    case $browser_type
      when :firefox, "firefox"
        sleep(sleep_amount.to_i + 3)
      when :chrome, "chrome"
        sleep(sleep_amount.to_i + 2)
      when :ie, :safari, "ie", "safari"
        if sleep_amount.to_i > 0
          sleep 3
        else
          sleep(sleep_amount.to_i * 3)
        end
      else
        fail "Which browser you usin' Willis?!"
    end
  end

  def instantiate_new_browser(site)
    if $runner == "watir"
      $newbrowser = Watir::IE.new
    elsif $runner == "watir-webdriver"
      $newbrowser = Watir::Browser.new $browsertype
    end
    $newbrowser.goto(site)
  end

  def go_to_new_tab
    $active_browser.windows.last.use
  end

  def get_browser_title
    $active_browser.title
  end

  def get_browser_url
    $active_browser.url
  end

  def search_browser_url(text)
    $active_browser.url.include? text
  end

  def maximize_browser
    if $runner == "watir"
      $active_browser.maximize
    elsif $runner == "watir-webdriver"
      $active_browser.window.maximize
    end
  end

  # def maximize_browser # this was used in the SOCS branch but not sure if it's really any different than the method above
  #   $active_browser.driver.manage.window.maximize
  # end

  def minimize_browser
    $active_browser.driver.manage.window.resize_to(480, 320)
  end

  def resize_browser(width, height)
    if $runner == "watir-webdriver" && $browsertype == :chrome
      $active_browser.window.resize_to(width, height)
    end
  end

  def go_back_in_browser
    $active_browser.back
  end

  def close_browser(options = {})
    if $runner == "watir" && options.empty?
      @browser.close
    elsif $runner == "watir-webdriver"
      if options.empty?
        @browser.driver.close
      elsif options[:title]
        @browser.window(:title => options[:title]).close
      elsif options[:url]
        @browser.window(:url => options[:url]).close
      end
    end
  end

  def close_new_tab
    $active_browser.windows.last.close
    sleep 1
    alert_click if alert_exists?
  end

  def refresh_browser
    $active_browser.refresh
  end

  def window_use_last
    $active_browser.windows.last.use
    bring_browser_to_front
  end

  def window_use(titlename)
    $active_browser.window(:title => titlename).use
  end

  def window_use_index(index)
    $active_browser.window(:index => index).use
  end

  def get_window_count
    $active_browser.windows.count
  end

  def attach_new_browser(handle)
    if $runner == "watir"
      newwindow = case
                    when options[:title]
                    then
                      Watir::IE.attach(:title, options[:title])
                    when options[:url]
                    then
                      begin
                        Watir::IE.attach(:url, options[:url])
                      rescue
                        Watir::IE.attach(:url, /#{options[:url]}/)
                      end
                    else
                      raise ArgumentError.new 'No option found for NewWindow.attachNewBrowser! Please specify which element to attach the browser with (e.g. :title => "Agent Center")'
                  end
      newwindow.speed = :fast
      newwindow.bring_to_front
      $active_browser = newwindow
      setup
    elsif $runner == "watir-webdriver"
      ### THIS ONLY WORKS WHEN YOU WANT TO ATTACH TO A WINDOW THAT WAS OPENED THROUGH AUTOMATION ###
      sleep(3)
      $active_browser.driver.switch_to.window($active_browser.driver.window_handles[handle])
    end
  end

  def alert_exists?
    $active_browser.alert.exists?
  end

  def alert_click
    $active_browser.alert.ok
  end

  def send_keys(key)
    $active_browser.send_keys(key)
    # require 'rautomation'
    # @modal = RAutomation::Window.new :title => /Security/
    # @modal.send_keys :tab
    # @modal.send_keys :tab
    # @modal.send_keys :enter
  end

  # ------ Validation / Error methods ------

  def error_exists?(b = $active_browser)
    connection_errors b
    cold_fusion_errors b
  end

  def connection_errors(b)
    unless b.frames.length > 0 && $runner == "watir-webdriver"
      ['there was an error processing your request.', 'Internal Server Error',
       'Error 500', 'Mach-II Exception', 'HTTP 404', 'Proxy Error',
       'The website cannot display the page', 'Service Temporarily Unavailable',
       "The resource you were attempting to\r\naccess is unavailable"].each { |e| b.text.include? e and process_failure(e) }
    end
  end

  def cold_fusion_errors(b)
    cf_errors = ['Error Occurred', 'Please excuse us ...']
    cf_message = 'Cold Fusion error'
    frame_count = b.frames.count
    unless b.frames && $runner == "watir-webdriver"
      cf_errors.each do |e|
        begin
          b.text.include?(e) and (process_failure(cf_message) and break)
        rescue FrameAccessDeniedException => m
          skip_frame(m) and break
        rescue WIN32OLERuntimeError => m
          skip_frame(m) and break
        else
          for i in 1..frame_count
            frame = b.frame(:index, i - 1)
            begin
              frame.text.include?(e) and (process_failure(cf_message) and break)
            rescue FrameAccessDeniedException => m
              skip_frame(m) and break
            rescue WIN32OLERuntimeError => m
              skip_frame(m) and break
            else
              begin
                frame_count_2 = frame.frames.count
              rescue FrameAccessDeniedException => m
                skip_frame(m) and break
              rescue WIN32OLERuntimeError => m
                skip_frame(m) and break
              else
                for i2 in 1..frame_count_2
                  frm = frame.frame(:index, i2 - 1)
                  begin
                    frm.text.include?(e) and (process_failure(cf_message) and break)
                  rescue FrameAccessDeniedException => m
                    skip_frame(m) and break
                  rescue WIN32OLERuntimeError => m
                    skip_frame(m) and break
                  else
                    begin
                      frame_count_3 = frm.frames.count
                    rescue FrameAccessDeniedException => m
                      skip_frame(m) and break
                    rescue WIN32OLERuntimeError => m
                      skip_frame(m) and break
                    else
                      for i3 in 1..frame_count_3
                        fm = frm.frame(:index, i3 - 1)
                        begin
                          fm.text.include?(e) and (process_failure(cf_message) and break)
                        rescue FrameAccessDeniedException => m
                          skip_frame(m) and break
                        rescue WIN32OLERuntimeError => m
                          skip_frame(m) and break
                        end
                      end
                    end
                  end
                end
              end
            end
          end
        end
      end
    end
  end

  def process_failure(message)
    fail "#{message}"
  end

  def skip_frame(exception)
    puts "Skipping frame due to #{exception}"
  end

  private :connection_errors, :cold_fusion_errors, :process_failure, :skip_frame

  # ------ PDF methods ------

  def pdf_temp(pdf_path)
    wsh = WIN32OLE.new('Wscript.Shell')
    sleep(2)
    wsh.AppActivate(/Adobe Reader/)
    $pdf_file_name = pdf_path
  end

  def open_pdf(pdf_name)
    @pdf_file_name = pdf_name
    @receiver = PDF::Reader::PageTextReceiver.new
  end

  def read_pdf(pdf_text_to_validate)
    PDF::Reader.open(@pdf_file_name) do |reader|
      reader.pages.each do |page|
        page.walk(@receiver)
        pdf_text = @receiver.content.to_s
        $pdf_found = true if pdf_text.include? pdf_text_to_validate
      end
    end
    fail "Text not found in PDF" unless $pdf_found
  end

  def frame_state(frame_type, frame_value)
    case $runner
      when "watir"
        !frame_type.is_a?(Symbol) || !frame_type.is_a?(String)
        !frame_value.is_a?(Symbol) || !frame_value.is_a?(String)
      when "watir-webdriver"
        frame_type.empty?
        frame_value.empty?
    end
  end

  def html_element_state(html_element)
    case $runner
      when "watir"
        !html_element.is_a?(Symbol) || !html_element.is_a?(String)
      when "watir-webdriver"
        html_element.empty?
    end
  end

  def extract_element(html_element)
    case html_element
      when :link, :links
        @element = :links
      when :button, :buttons
        @element = :buttons
      when :checkbox, :checkboxes
        @element = :checkboxes
      when :radio, :radios
        @element = :radios
      when :span, :spans
        @element = :spans
      when :td, :tds
        @element = :tds
      else
        fail "Element type not defined - specify link or button, or add additional code to accommodate the element type"
    end
  end

  def set_operation(html_element)
    case html_element.to_sym
      when :text_field, :radio, :checkbox, :textarea then
        :set
      when :select_list then
        :select
      else
        fail "HTML element not recognized"
    end
  end

  def set_html_element(html_element)
    case html_element.to_sym
      when :text_field, :text_fields then
        :text_fields
      when :input, :inputs then
        :inputs
      when :radio, :radios then
        :radios
      when :select_list, :select_lists then
        :select_lists
      when :checkbox, :checkboxes then
        :checkboxes
      when :button, :buttons then
        :buttons
      else
        fail "HTML element not recognized"
    end
  end

  # ------ Text methods ------

  def text_exists?(text, html_element = {}, html_element_type = {}, html_element_value = {})
    if html_element_state(html_element)
      if text.is_a? Regexp
        txt = text.match $active_browser.text
        txt.nil? ? false : true
      else
        $active_browser.text.include? text
      end
    else
      find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).text.include? text
    end
  end

  # For arrays only
  def text_exists_within_element?(text_to_confirm, watir_element)
    watir_element.inner_html.include?(text_to_confirm.to_s)
  end

  def text_exists_in_frame?(text, frame_type = {}, frame_value = {}, html_element = {}, html_element_type = {}, html_element_value = {})
    if html_element_state(html_element)
      find_element(:frame, frame_type.to_sym, frame_value).text.include? text
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).text.include? text
    end
  end

  ##########################################  Wait Methods  ##########################################
  def wait_until_element_exists(tag, attribute_name, attribute_value)
    $active_browser.wait_until { element_exists?(tag, attribute_name, attribute_value) }
  end

  def wait_until_element_visible(tag, attribute_name, attribute_value)
    $active_browser.wait_until { element_visible?(tag, attribute_name, attribute_value) }
  end

  def wait_until_element_enabled(tag, attribute_name, attribute_value)
    $active_browser.wait_until { element_enabled?(tag, attribute_name, attribute_value) }
  end

  # ------ Set element method ------

  def set_element(html_element, html_element_type, html_element_value, field_value, frame_type = {}, frame_value = {})
    html_element = :text_field if html_element == :input
    if frame_state(frame_type, frame_value)
      if html_element == :radio
        find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).set
      else
        find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).send(set_operation(html_element), field_value)
      end
    else
      if html_element == :radio
        find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).set
      else
        find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).send(set_operation(html_element), field_value)
      end
    end
  end

  def set_element_within_element(html_element1, html_element_type1, html_element_value1, html_element2, html_element_type2, html_element_value2, field_value, frame_type = {}, frame_value = {})
    html_element1 = :text_field if html_element1 == :input
    if frame_state(frame_type, frame_value)
      if html_element1 == :radio
        find_element(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).set
      else
        find_element(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).send(set_operation(html_element1), field_value)
      end
    else
      if html_element1 == :radio
        find_element(:frame, frame_type.to_sym, frame_value).send(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).set
      else
        find_element(:frame, frame_type.to_sym, frame_value).send(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).send(set_operation(html_element1), field_value)
      end
    end
  end

  def set_element_by_index(html_element, html_element_type, html_element_value, index, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      if html_element == :select_list
        fail "Dropdown does not have #{index} options" if index > find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).options.count-1
        find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).options.each do |s|
          for i in 0..find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).options.count-1
            if s.index == index
              value = s.text
              find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).send(:select, value)
              return
            end
          end
        end
      else
        fail "Method: 'set_element_by_index' only coded for select lists"
      end
    else
      if html_element == :select_list
        fail "Dropdown does not have #{index} options" if index > find_element(:frame, frame_type.to_sym, frame_value).send(html_element, html_element_type, html_element_value).options.count-1
        find_element(:frame, frame_type.to_sym, frame_value).send(html_element, html_element_type, html_element_value).options.each do |s|
          for i in 0..find_element(:frame, frame_type.to_sym, frame_value).send(html_element, html_element_type, html_element_value).options.count-1
            if s.index == index
              value = s.text
              find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).send(:select, value)
              return
            end
          end
        end
      else
        fail "Method: 'set_element_by_index' only coded for select lists"
      end
    end
  end

  def set_checkbox(html_element_type, html_element_value, operation, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(:checkbox, html_element_type.to_sym, html_element_value).set(operation)
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).set(operation)
    end
  end

  def set_checkbox_within_element(html_element_type1, html_element_value1, html_element2, html_element_type2, html_element_value2, operation, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(:checkbox, html_element_type1.to_sym, html_element_value1).set(operation)
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(:checkbox, html_element_type1.to_sym, html_element_value1).set(operation)
    end
  end

  def set_visible_element(html_element, html_element_type, html_element_value, field_value)
    for i in 0..find_element(set_html_element(html_element), html_element_type.to_sym, html_element_value).count-1
      if find_element(set_html_element(html_element), html_element_type.to_sym, html_element_value)[i].visible?
        if html_element == :radio
          find_element(set_html_element(html_element), html_element_type.to_sym, html_element_value)[i].send(set_operation(html_element))
        else
          find_element(set_html_element(html_element), html_element_type.to_sym, html_element_value)[i].send(set_operation(html_element), field_value)
        end
        break
      end
    end
  end

  ##########################################  Set Field Value Methods  ##########################################
  def set_html_content(tag, attribute_name, attribute_value, content)
    find_element(tag, attribute_name, attribute_value).set(content)
  end

  def select_list_set_value(tag, attribute_name, attribute_value, content)
    find_element(tag, attribute_name, attribute_value).select_value(content)
  end

  ##########################################  Dropdown Methods  ##########################################
  def select_list_set_value_by_text(attribute_name, attribute_value, content)
    $active_browser.select_list(attribute_name, attribute_value).options.each { |option|
      if option.text.include? content
        find_element(:select_list, attribute_name, attribute_value).select(option.text)
        break
      end
    }
  end

  def select_list_set_value_by_index(attribute_name, attribute_value, index_value)
    if index_value
      $active_browser.send(:select_list, attribute_name, attribute_value).options[index_value.to_i].click
    end
  end

  def set_visible_select_list_by_index(html_element_type, html_element_value, index_value)
    for i in 0..find_element(set_html_element(:select_list), html_element_type.to_sym, html_element_value).count-1
      if find_element(set_html_element(:select_list), html_element_type.to_sym, html_element_value)[i].visible?
        find_element(set_html_element(:select_list), html_element_type.to_sym, html_element_value)[i].options[index_value.to_i].click
        break
      end
    end
  end

  def select_list_retrieve_exact_value(attribute_name, attribute_value, similar_content)
    $active_browser.select_list(attribute_name, attribute_value).options.each { |option|
      if option.text=== similar_content
        find_element(:select_list, attribute_name, attribute_value).select(option.text)
        return option.text
      end
    }
    return false
  end

  def select_list_check_selected(attribute_name, attribute_value, selectedOption)
    $active_browser.select_list(attribute_name, attribute_value).options.each { |option|
      if option.text === selectedOption
        if option.selected? === false
          fail "#{selectedOption} was not the selected option"
        else
          return true
        end
        break
      end
    }
    fail "#{selectedOption} was not in the list of options"
  end

  def select_list_contains(attribute_name, attribute_value, value)
    $active_browser.select_list(attribute_name, attribute_value).options.each { |option|
      if option.text == value
        return true
      end
    }
    fail "#{value} not found in dropdown"
  end

  def select_list_contains?(attribute_name, attribute_value, value)
    $active_browser.select_list(attribute_name, attribute_value).options.each { |option|
      if option.text == value
        return true
      end
    }
  end

  def select_list_not_contains(attribute_name, attribute_value, value)
    $active_browser.select_list(attribute_name, attribute_value).options.each { |option|
      if option.text == value
        return false
      end
    }
  end

  def select_list_includes(attribute_name, attribute_value, value)
    $active_browser.select_list(attribute_name, attribute_value).options.each { |option|
      puts option.text
      puts value
      if option.text.include? value
        return true
      end
    }
  end

  def select_list_correct_contents(attribute_name, attribute_value, array_of_dropdown_contents)
    @index = 0
    while @index < array_of_dropdown_contents.length
      $active_browser.select_list(attribute_name, attribute_value).options.each { |option|
        if option.text == array_of_dropdown_contents[@index]
          #puts array_of_dropdown_contents[@index] + " vs " + option.text
          return true
        end
        #puts array_of_dropdown_contents[@index] + " vs " + option.text
      }
      fail "#{array_of_dropdown_contents[@index]} not found in dropdown"
      return false
      @index = @index + 1
    end
  end

  def select_list_excludes_content(attribute_name, attribute_value, array_of_dropdown_contents)
    @index = 0
    while @index < array_of_dropdown_contents.length
      $active_browser.select_list(attribute_name, attribute_value).options.each { |option|
        if option.text != array_of_dropdown_contents[@index]
          return true
        else
          fail "#{option} was found in the dropdown"
          return false
        end
      }
      @index = @index + 1
    end
  end

  def select_list_not_includes(attribute_name, attribute_value, value)
    $active_browser.select_list(attribute_name, attribute_value).options.each { |option|
      # puts "(Option) found #{option.text} vs (value) #{value}"
      if option.text.include? value
        fail "(Option) found #{option.text} vs (value) #{value}"
        return true
      end
    }
  end

  # ------ Get element methods ------

  def get_element_value(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).value
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).value
    end
  end

  def get_element_value_within_element(html_element1, html_element_type1, html_element_value1, html_element2, html_element_type2, html_element_value2, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).value
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).value
    end
  end

  def get_element_text(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).text
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).text
    end
  end

  def get_element_text_within_element(html_element1, html_element_type1, html_element_value1, html_element2, html_element_type2, html_element_value2, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).text
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).text
    end
  end

  def get_element_selected_option(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).selected_options.each do |option|
        option.text
      end
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).selected_options.each do |option|
        option.text
      end
    end
  end

  def get_element_inner_html(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).inner_html
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).inner_html
    end
  end

  def get_element_length(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).length
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).length
    end
  end

  def get_element_length_within_element(html_element1, html_element_type1, html_element_value1, html_element2, html_element_type2, html_element_value2, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).length
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).length
    end
  end

  def get_element_count(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    extract_element(html_element)
    if frame_state(frame_type, frame_value)
      find_element(@element.to_sym, html_element_type.to_sym, html_element_value).count
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(@element.to_sym, html_element_type.to_sym, html_element_value).count
    end
  end

  def get_element_attribute(html_element_type, attribute_name, attribute_value, attribute_type_to_find)
    find_element(html_element_type, attribute_name, attribute_value).attribute_value(attribute_type_to_find)
  end

  def get_html_content(tag, attribute_name, attribute_value, desired_content)
    find_element(tag, attribute_name, attribute_value).send(desired_content)
  end

  def get_html_content_by_index(tag, attribute_name, attribute_value, desired_content, index)
    find_element(tag, attribute_name, attribute_value)[index].send(desired_content)
  end

  def get_html_element(parent_type, parent_attribute, parent_attribute_value)
    find_element(parent_type, parent_attribute, parent_attribute_value)
  end

  # ------ Fire event method ------

  def element_fire_event(html_element, html_element_type, html_element_value, event, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).send(:fire_event, event)
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).send(:fire_event, event)
    end
  end

  # ------ Generic element method ------

  def element_do(html_element, html_element_type, html_element_value, action, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).send(action)
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).send(action)
    end
  end

  def find_element(tag, attribute_name, attribute_value)
    if $app
      if $app == "CIQ"
        Watir::Wait.until {
          # puts "Modal exists #{$active_browser.div(class: 'modal-mini-loading').exists?}"
          # puts "Waiting... for #{tag} - #{attribute_name} - #{attribute_value}"
          !$active_browser.div(class: 'modal-mini-loading').exists?
        }
      end
    end
    $active_browser.send(tag, attribute_name, attribute_value)
  end

  # ------ Click element method ------

  def click_element(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    attempts ||= 3
    begin
      begin
        if frame_state(frame_type, frame_value)
          find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).click
        else
          find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).click
        end
      rescue
        element_fire_event(html_element, html_element_type, html_element_value, :onclick, frame_type, frame_value)
      end
    rescue
      (attempts-=1).zero? ? raise("#{html_element_value} not found on screen") : retry
    end
  end

  def click_element_then_sleep(html_element, html_element_type, html_element_value, sleep_time, frame_type = {}, frame_value = {})
    begin
      if frame_state(frame_type, frame_value)
        find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).click
      else
        find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).click
      end
    rescue
      element_fire_event(html_element, html_element_type, html_element_value, :onclick, frame_type, frame_value)
    end
    sleep sleep_time
  end

  def click_visible_element(html_element, html_element_type, html_element_value)
    for i in 0..find_element(set_html_element(html_element), html_element_type.to_sym, html_element_value).count-1
      if find_element(set_html_element(html_element), html_element_type.to_sym, html_element_value)[i].visible?
        find_element(set_html_element(html_element), html_element_type.to_sym, html_element_value)[i].click
        break
      end
    end
  end

  def click_element_within_element(html_element1, html_element_type1, html_element_value1, html_element2, html_element_type2, html_element_value2, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).click
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).click
    end
  end

  def click_element_by_occurrence(html_element, html_element_type, html_element_value, occurrence, frame_type = {}, frame_value = {})
    occurrence = occurrence.to_i
    occurrence = occurrence-1
    extract_element(html_element)
    if frame_state(frame_type, frame_value)
      find_element(@element.to_sym, html_element_type.to_sym, html_element_value)[occurrence].click
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(@element.to_sym, html_element_type.to_sym, html_element_value)[occurrence].click
    end
  end

  def click_link_in_sorted_table(table_attribute, table_attribute_name, tag, tag_attribute, link)
    for i in 0..find_element(:table, table_attribute, table_attribute_name).send(tag, tag_attribute, link).count
      if find_element(:table, table_attribute, table_attribute_name).send(tag, tag_attribute, link)[i].visible?
        find_element(:table, table_attribute, table_attribute_name).send(tag, tag_attribute, link)[i].click
        break
      end
    end
  end

  # ------ Focus element method ------

  def focus_element(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).focus
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).focus
    end
  end

  # ------ Hover element method ------

  def hover_element(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).hover
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).hover
    end
  end

  # ------ Element exists? method ------

  def element_exists?(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).exists?
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).exists?
    end
  end

  def element_exists_within_element?(html_element1, html_element_type1, html_element_value1, html_element2, html_element_type2, html_element_value2, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).exists?
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).exists?
    end
  end

  # ------ Element visible? method ------

  def element_visible?(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      begin
        find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).visible?
      rescue
        return false
      end
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).visible?
    end
  end

  def element_present?(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      begin
        find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).present?
      rescue
        return false
      end
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).present?
    end
  end

  def element_not_visible?(tag, attribute_name, attribute_value)
    begin
      if find_element(tag, attribute_name, attribute_value).visible?
        return false
      else
        return true
      end
    rescue
      return true
    end
  end

  def element_visible_within_element?(html_element1, html_element_type1, html_element_value1, html_element2, html_element_type2, html_element_value2, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).visible?
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).visible?
    end
  end

  def element_visible_in_sorted_table?(html_element, html_element_type, html_element_value, table_attribute, table_attribute_name)
    extract_element(html_element)
    for i in 0..find_element(:table, table_attribute, table_attribute_name).send(@element, html_element_type, html_element_value).count
      if find_element(:table, table_attribute, table_attribute_name).send(@element, html_element_type, html_element_value)[i].visible?
        break
      end
    end
  end

  def element_visible_by_occurrence?(html_element, html_element_type, html_element_value, occurrence, frame_type = {}, frame_value = {})
    occurrence = occurrence.to_i
    occurrence = occurrence-1
    set_html_element(html_element)
    if frame_state(frame_type, frame_value)
      find_element(set_html_element(html_element), html_element_type.to_sym, html_element_value)[occurrence].visible?
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(set_html_element(html_element).to_sym, html_element_type.to_sym, html_element_value)[occurrence].visible?
    end
  end

  # ------ Element disabled? method ------

  def element_disabled?(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).disabled?
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).disabled?
    end
  end

  def element_disabled_within_element?(html_element1, html_element_type1, html_element_value1, html_element2, html_element_type2, html_element_value2, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).disabled?
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).disabled?
    end
  end

  # ------ Element enabled? method ------

  def element_enabled?(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).enabled?
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).enabled?
    end
  end

  def element_enabled_within_element?(html_element1, html_element_type1, html_element_value1, html_element2, html_element_type2, html_element_value2, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).enabled?
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).enabled?
    end
  end

  # ------ Element checked? method ------

  def element_checked?(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).checked?
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).checked?
    end
  end

  def element_checked_by_occurrence?(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    occurrence = occurrence.to_i
    occurrence = occurrence-1
    set_html_element(html_element)
    if frame_state(frame_type, frame_value)
      find_element(set_html_element(html_element), html_element_type.to_sym, html_element_value)[occurrence].checked?
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(set_html_element(html_element).to_sym, html_element_type.to_sym, html_element_value)[occurrence].checked?
    end
  end

  def element_checked_within_element?(html_element1, html_element_type1, html_element_value1, html_element2, html_element_type2, html_element_value2, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).checked?
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).checked?
    end
  end

  # ------ Element read-only? method ------

  def element_readonly?(html_element, html_element_type, html_element_value, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element.to_sym, html_element_type.to_sym, html_element_value).readonly?
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element.to_sym, html_element_type.to_sym, html_element_value).readonly?
    end
  end

  def element_readonly_within_element?(html_element1, html_element_type1, html_element_value1, html_element2, html_element_type2, html_element_value2, frame_type = {}, frame_value = {})
    if frame_state(frame_type, frame_value)
      find_element(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).readonly?
    else
      find_element(:frame, frame_type.to_sym, frame_value).send(html_element2.to_sym, html_element_type2.to_sym, html_element_value2).send(html_element1.to_sym, html_element_type1.to_sym, html_element_value1).readonly?
    end
  end

  # ------ Browser Methods -------

  # returns first element that matches the id_attr of attr_value
  def return_html_element(browser_method, id_attr, attr_value)
    method_to_call = $active_browser.method(browser_method)
    method_result = method_to_call.call
    if method_result.respond_to? :each
      method_result.each do |html_element|
        if html_element.send(id_attr) == attr_value
          return html_element
        end
      end
    end
    fail "The method #{browser_method} did not return an iterable result to find the matching element."
  end

  # ------ General methods ------

  def values_equal?(value1, value2)
    puts "--testing: value1(#{value1}) = value2(#{value2}); #{value1 == value2}"
    value1 == value2
  end

  def find_element_by_regex_id?(html_element, regex)
    case html_element
      when :label, "label"
        element = :label
      when :span, "span"
        element = :span
      when :select_list, "select_list", "select list", "dropdown"
        element = :select_list
      else
        fail "Html element not recognized"
    end
    find_element(element, :id, regex)
  end

  def run_script(script)
    $active_browser.execute_script(script)
  end

  def fail_unless(begin_stmt, rescue_stmt, time = {})
    sleep(1)
    begin
      if time.nil?
        Watir::Wait.until { begin_stmt }
      else
        Watir::Wait.until(time.to_i) { begin_stmt }
      end
    rescue
      if rescue_stmt =~ /fail/
        stmt_length = rescue_stmt.length
        fail_stmt = fail rescue_stmt.to_str[5, stmt_length.to_i]
        return fail_stmt
      else
        rescue_stmt
      end
    end
  end

  def get_data_from_yml(filename, desired_data)
    data = YAML.load_file(filename)
    data[desired_data] ? found = true : found = false
    unless found
      fail "no data found in '#{filename}' for '#{desired_data}'"
    end
    data[desired_data]
  end


  def get_userid_password(userid)
    @password_file = "password"

    def get_userid_password_filename
      @password_file == "password" ? filename = ENV['HOME'] + "\\Documents\\passwords.yml" : filename = @password_file
      return filename
    end

    filename = get_userid_password_filename
    unless File.exists?(filename)
      # See the instructions above on what to do if you get this error
      raise ArgumentError.new("USER ERROR: Please see the comments for this method for instructions on how to create file #{filename}")
    end
    data = YAML::load(File.read(filename))
    data[userid] == nil ? data["default"] : data[userid]
  end

end
