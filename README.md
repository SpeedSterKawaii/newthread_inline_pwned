# newthread_inline_pwned

newthread with one singular offset.  This function copies the newthread chunk from memory, fixes the RVA values, and fires it with an assembly stub.

```cpp
template < typename state_t >
state_t new_thread( const state_t state )
{
	constexpr auto lua_newthread_function_size = 0x1CE;

	constexpr auto call_size = sizeof( std::uint8_t ) + sizeof( std::uintptr_t );

	static const auto roblox_base = reinterpret_cast< std::uintptr_t >( GetModuleHandleA( nullptr ) );

	static const auto lua_newthread_clone = VirtualAlloc( nullptr, lua_newthread_function_size + sizeof( std::uint8_t ), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE );

	static const auto lua_newthread_function = roblox_base + 0x3E8D54;

	std::memcpy( lua_newthread_clone, reinterpret_cast< const void* >( lua_newthread_function ), lua_newthread_function_size );
	
	*reinterpret_cast< std::uint8_t* >( reinterpret_cast< std::uintptr_t >( lua_newthread_clone ) + lua_newthread_function_size ) = 0xC3;

	for ( auto iterator = 0u; iterator < lua_newthread_function_size; ++iterator )
	{
		const auto lua_newthread_clone_absolute = reinterpret_cast< std::uintptr_t >( lua_newthread_clone ) + iterator;

		if ( *reinterpret_cast< std::uint8_t* >( lua_newthread_clone_absolute ) == 0xE8 )
		{
			const auto original_function_location = *reinterpret_cast< std::uintptr_t* >( ( lua_newthread_function + iterator ) + sizeof( std::uint8_t ) ) + ( lua_newthread_function + iterator ) + call_size;

			*reinterpret_cast< std::uintptr_t* >( lua_newthread_clone_absolute + sizeof( std::uint8_t ) ) = original_function_location - lua_newthread_clone_absolute - call_size;
		}
	}

	state_t thread;

	__asm
	{
		push finish;
		mov eax, state;
		jmp lua_newthread_clone;

	finish:
		mov thread, eax;
	}

	return thread;
}
```
