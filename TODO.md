* Fix frames vs Hz issue
* No more tabs
* Multi unit activity
* Refactor 'receptive_field' logic to account for all 3 cases (will involve more complex dimension checks)
* Add compute receptive field, encoder, decoder logic
* Plot receptive fields (similar to stimuli but needs a special unit selector)
* Save computed arrays
* Screenshots
* Document how our methods work for smaller kernels but can break if they're too large relative to length of the recording due to padding
