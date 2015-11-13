# vim: ft=ruby
guard 'shell' do
  watch('README.md') do
    puts "#{Time.now}: Updated"
    `slidybuild < README.md > index.html`
  end
end
